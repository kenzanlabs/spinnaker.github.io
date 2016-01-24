---
layout: toc-page
title: How do I configure Spinnaker
id: properties
lang: en
---

* Table of contents. This line is required to start the list.
{:toc}

# What is this document about?

See [Custom Configuring a Spinnaker Deployment](configuration.html) for a
discussion of Spinnaker configuration as presented by Spinnaker.

This document is an alternative perspective from the Spring implementation
point of view.


# How do I configure Spinnaker?

When you setup Spinnaker according to direction at this site, you will get a set of [configuration files](https://github.com/spinnaker/spinnaker/tree/master/config). Spinnaker comes with `spinnaker.yml`, `spinnaker-local.yml`, and a `<module>.yml` file (for each module), intent on simplifying configuration.

Each Spinnaker module is coded to look for these config files here:

* `$HOME/.spinnaker` - This is coded into the application, and the place for development
* `/opt/spinnaker/config` - Spinnaker's packaging scripts bundle each module up into a debian package for installation on any machine. They plugin this override path.
* `/opt/<module>/config` - Another optional place the property files are installed.

Here is a description of how these files work together:

1. `spinnaker.yml` is the starting point for property settings. It isn't the place to plug in overrides, so we won't go into any detail.

2. `spinnaker-local.yml` gives you the place to override settings (like enabling providers and services) and supply needed values (like a Jenkins base URL that Spinnaker has no way of knowing). The comments should help you navigate these various settings.

3. `<module>.yml` is used to configure each module. By default, they are not meant for direct editing except for certain use cases.

To see some use cases for overrides, see below.

# Use Cases

Here are some situations you may run into and a way to work through it. NOTE: It's possible to combine use cases.

## Use case 1: Can I stick my super secret credentials in there somewhere?

Try to avoid that!! Secrets are meant to stay secret. Instead, see if you can either supply credentials through environment variables, or configure a separate property file that isn't put under source control.

## Use case 2: &lt;module>.yml just doesn't cut it

An excellent example is `clouddriver.yml` which lists accounts to talk to. In Cloud Foundry, if you have more than one account to wire, you'll want to create a `clouddriver-local.yml` file to override the single account setting provided by default. Local is a profile automatically added, giving you space to write an override.

## Use case 3: I need my config files somewhere unique

Put them where you need them and run things with `spring.config.location` (or `SPRING_CONFIG_LOCATION`) pointed at your alternate path.

## Use case 4: Property inheritance doesn't fit my needs

At the end of the day, you can simply write your own set of module-specific YAML files and not reference any properties from `spinnaker.yml`.

# How Spinnaker resolves properties

Spinnaker is highly customizable thanks to externalized property settings. This means that the various URLs, ports, and configuration settings to talk to your Spinnaker environment as well as the place your deployments are sent, are adjustable.

This is done using Spring Boot's [externalized property settings](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config).

Look at the following code from Spinnaker:

{% highlight java %}
@Component
@ConditionalOnProperty('slack.enabled')
class SlackNotificationService implements NotificationService {
  private static Notification.Type TYPE = Notification.Type.SLACK

  @Autowired
  SlackService slack

  @Value('${slack.token}')
  String token
  ...
{% endhighlight %}

Assume that you are configuring echo.

There are two properties listed: `slack.enabled` and `slack.token`. These are properties that dictate whether or not this part of [echo](https://github.com/spinnaker/echo) is enabled or not, and if so, what the token value is to connect it to a slack webhook.

The `@Value` annotation is a core component of the Spring Framework that has been available for years, and `@ConditionalOnProperty` is from Spring Boot, allowing components, services, etc. to optionally by activated.

Spring Boot has an order it follows when trying to resolve property values.

1. It looks for a default value right in the annotation. `@Value("${slack.token:my-default-value}") would be the way to specify **my-default-value** in the code.

2. Spring Boot will look for overrides in the application's `echo.yml` and `spinnaker.yml` files that are bundled up in its JAR file. 

 > **NOTE:** Spring Boot actually looks for `application.yml` and `application-${profile}.yml` files. But the value `application` is overridden in Spinnaker using `spring.config.name=echo,spinnaker`. This lets each module put all their configuration files in one folder without conflict while also reading from a common superset of properties in `spinnaker.yml` (and its variants).

3. Spring Boot will then look for any overrides tied to a specific **profile**. Profile-specific property files look like `echo-${profile}.yml`.

 > **QUESTION:** What's a profile? A profile is a collection of properties tied to a specific environment. For example, you could have **dev**, **test**, and **production**. You can have more than one profile active at any given time. These are activated by setting `spring-cloud-starter-eurekafiles.active`. For example you could apply `spring.profiles.active=cloud,production` to your setup. That would cause echo, in this situation, to look for `echo-cloud.yml` and `echo-production.yml` for options to override any previous properties.

4. By default, Spring Boot looks inside the JAR file as well as adjacent to the JAR file for property files. But you can also tell Boot to look in other places by overriding `spring.config.location`. Spinnaker has each module configured to look in `$HOME/.spinnaker` in its source code, and the build scripts provide other options, such as `/opt/spinnaker/config`. If you are setting up Spinnaker and want to locate all your configuration settings somewhere else, just override that setting.

5. Spring Boot will also look for any environment variables that override property settings by inspecting environment variables. Environment variables can be supplied to a JVM process using `-D`. For example, `java -Dslack.token=secret -jar myapp.jar` would feed Spring Boot an environmental override for that property. To make things flexible for multiple platforms, Spring Boot supports [relaxed bindings](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config-relaxed-binding). This means that `SLACK_TOKEN` and `slack.token` will both resolve to `@Value("${slack.token}")`. Hence, `SLACK_TOKEN=secret java -jar myapp.jar` will also work. 

 > **NOTE**: Environment variables are handy for certain environments like Cloud Foundry where you may not have access to either the local filesystem to drop an adjacent property file nor be able to add an extra argument to the JRE. Cloud Foundry allows you to apply environment properties to any application, giving you the power to override any property.

# Why is this so hard?!?

Well, you are talking about linking together multiple processes in a highly customizable way. This mechanism lets the Spinnaker team roll out a base configuration while still giving you maximum power to tweak every single setting. The trade off for such flexibility is having to embrace all these moving parts. If you find any issues, [let us know](https://github.com/spinnaker/spinnaker/issues).