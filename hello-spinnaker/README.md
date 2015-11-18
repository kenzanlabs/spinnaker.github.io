#An Introduction to Spinnaker: Hello Deployment

This guide will run through the workflow of setting up an example application deployment with [Spinnaker](http://spinnaker.io/). It assumes you already have Spinnaker up and running on AWS or GCE. A guide for installation can be found [here](http://spinnaker.io/documentation/getting_started.html).

Below is a diagram of the workflow we will setup.

![diagram](flow.png)

##Setup Jenkins

Jenkins is a powerful continuous integration server that allows us to several important things:

* Poll our Github repository for changes so Spinnaker knows when to run a pipeline.
* Compile and build our example application into .deb package so Spinnaker can bake it on an image. Spinnaker expects all applications to be deployed as deb packages.
* Publish our application .deb to a our repository (see below)
* Hosts a private deb repository. This allows Spinnaker to `apt-get install` our application when the image is baked.

###Installing Jenkins

If you already have a Jenkins server, you can skip this step. However be sure that port 9999 on the server is open to the internet, and that your Jenkins port is accessible by your Spinnaker instance.

Begin by launching a new Ubuntu Trusty image on your cloud, as mentioned above be sure that port 9999 is open and Jenkins port 8080 is accessible to you and your Spinnaker ip.

SSH into your instance and run the following:

```
# apt-get update
# apt-get upgrade
# apt-get install openjdk-7-jdk
# wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
# sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
# apt-get update
# apt-get install jenkins
# service jenkins start
```

Visit port :8080 on your instance and you should see Jenkins startup and present the dashboard.

###Enable Jenkins API

Spinnaker communicates with Jenkins by using it's REST API. However the API is not enabled by default. To enable it we first must enable global security (which is a good idea anyway). Click "Manage Jenkins" then "Configure Global Security". Under Access Control check "Jenkins own database" and "allow users to sign up". Under authorization check "Logged-in users can do anything" and save. 

![jenkins1](jenkins1.png) 

You will now be presented with a login screen. Click the "Create an account" link to register. Take note of your username and password, Spinnaker will need them later. Now you can turn off allowing users to register as you can add them manually from the control panel. The Jenkins API is now enabled.

###Setup dep repo (Aptly)

[Aptly](http://www.aptly.info/) is a tool that easily allows us to manage and publish the packages that Jenkins will build. After we install and configure the tool, nginx will host the repository on our jenkins server so it can be consumed during the bake process.

The following will install aptly to the jenkins user home directory so our jobs can easily use it. We will also create our repo so the jobs can easily add packages.

```
$ mkdir /opt/aptly && cd /opt/aptly
$ wget https://dl.bintray.com/smira/aptly/0.9.5/debian-squeeze-x64/aptly
$ chmod +x aptly
```

Create our repo named "hello"

```
deb-s3 upload --bucket spinnaker-debs hello-karyon-rxnetty_1-1_all.deb
deb-s3 upload --bucket spinnaker-debs --arch amd64 --preserve-versions true  hello-karyon-rxnetty_1-1_all.deb
deb-s3 upload --bucket spinnaker-debs --arch amd64 --preserve-versions true *.deb
$ ./aptly repo create hello
```

We will now publish our (currently empty) repo and setup nginx to host it on port 9999.

```
# ./aptly publish repo -architectures="amd64" -component=main -distribution=trusty -skip-signing=true hello
```
###Setup Nginx to serve deb repo
Install and configure nginx

```
# apt-get install nginx
# vim /etc/nginx/sites-enabled/default
```

Contents of /etc/nginx/sites-enabled/default:

```
server {
        listen 9999 default_server;
        listen [::]:9999 default_server ipv6only=on;
        root home/ubuntu/.aptly/public;
        index index.html index.htm;
        server_name localhost;
        location / {
                try_files $uri $uri/ =404;
        }
}
```

###Setup dep repo (debs3)

Start nginx `service nginx start`

###Fork example application

We have set up an example application with a gradle.build ready to package your app for spinnaker deployment. Fork [https://github.com/kenzanlabs/hello-karyon-rxnetty](https://github.com/kenzanlabs/hello-karyon-rxnetty) via the github UI so you can make changes and see them flow through the Spinnaker pipeline.

###Create Jenkins Jobs

We will now setup our 4 jenkins jobs for spinnaker to use.

1. Polling Job

The first is a simple job that polls our git repo for changes. Spinnaker has no knowledge of our repo location, so it needs a way to trigger a pipeline automatically when code is pushed. (A pipeline is a set of actions that handle the application delivery) Spinnaker will poll our polling job and kick off a pipeline when it detects a fresh run.

* Ensure the Jenkins git plugin is enabled and create a new Freesyle Project named "Example Poll Github".

![diagram](jenkins2.png)

* Add your app fork git address.
* Under build triggers check "Poll SCM" and enter "* * * * *" for Jenkins to poll once a minute. You can now save the job.

##Configure Spinnaker

SSH to your Spinnaker instance by tunneling the needed ports. Tunneling ensures that Spinnaker is not accessible from the internet outside of your ssh connection. This is **very important** because anyone who has access to Spinnaker can do anything your cloud account can do.

The ports needed:

* 9000: html/js UI (Deck)
* 8084: API entrypoint (Gate)
* 8087: Image bakery (Rosco)

```
ssh -i yourkey.pem -L 9000:127.0.0.1:9000 8084:127.0.0.1:8084 8087:127.0.0.1:8087 ubuntu@instanceip
```

Spinnaker will be accessible on [http://localhost:9000](http://localhost:9000).

###Jenkins Integration

Before we can begin setting up our workflow, we need to edit the Spinnaker configuration to allow it to communicate with our Jenkins server.

Begin by shutting down Spinnaker. Spinnaker configuration is stored in memory so we need to reload it everytime we make a change. You can now edit the master configuration file:

```
# stop spinnaker
# vim /opt/spinnaker/config/spinnaker-local.yml
```

Scroll down to jenkins section and add your url, username and password from above

```
  jenkins:
    defaultMaster:
      name: Jenkins # The display name for this server
      baseUrl: http://jenkins-server:8080
      username: jenkinsuser
      password: jenkinspassword
```

You also need to enable igor, the service that communicates with the jenkins api.

```
igor:
    enabled: true
```

###Deb Repository

The last configuration step is to add our deb repository address to the Rosco config. When Spinnaker is baking the application image, it will add this address to the sources list so it can `apt-get install` the deb package. For this reason it is important that port 9000 is open on our Jenkins server which hosts the repo.

```
# vim /opt/rosco/config/rosco.yml
```
Add your jenkins server address and repo port to the debianRepository key

```
debianRepository: http://jenkins-server:9999
```
We can now start Spinnaker and start configuring our workflow. `# start spinnaker`
