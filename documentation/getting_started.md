---
layout: toc-page
title: Getting Started
id: getting_started
lang: en
---

* Table of contents. This line is required to start the list.
{:toc}

# Getting started

These instructions quickly get Spinnaker up and running on an Amazon
Web Services (AWS) EC2 or Google Compute Engine (GCE) virtual machine
(VM) or local machine, then walk you through the steps of setting up
and using Spinnaker to deploy to and manage clusters on AWS and/or
GCE.

You will go through the following steps:

1. Setup a project representing your target deployment
environment. This project will house the clusters that are deployed
and managed by Spinnaker.
1. Setup a virtual machine that will be used to run Spinnaker.
1. Install, configure and run Spinnaker.
1. Configure an example Spinnaker pipeline to bake an image and deploy
the image to a cluster, and let you explore Spinnaker in operation.
1. When you're done experimenting, clean everything up and stop Spinnaker.

The first two steps, which have very little to do with Spinnaker
itself, are by far the most complex steps.

Keep in mind that naming of your entities in AWS is important as these names are parsed
and Spinnaker will use them to populate available resource lists in the Spinnaker
UI.

## Step 1: Set up your target deployment environment

You need to setup your target deployment environment, which is an AWS
or GCP project that will house clusters that are deployed to and
managed by Spinnaker. You can choose to setup only an [AWS
environment](#aws-setup), a [GCP
environment](#google-cloud-platform-setup), or both. Please
follow the appropriate instructions below to get your target
deployment environment(s) setup.

### AWS Setup

If you'd like to have Spinnaker deploy to and manage clusters on AWS,
you'll need to have an AWS project setup. If you've already got one,
please skip to the next step. Otherwise, please follow the
instructions below.

Sign into the [AWS console](https://console.aws.amazon.com) and let
AWS pick a default region where your project resources will be
allocated. In the rest of this tutorial, we'll assume that the region
assigned is <code>us-west-2</code>. If the region selected for your
project is different from this, please substitue your region
everywhere <code>us-west-2</code> appears below.

Also, in the instructions below, we'll assume that your AWS account
name is <code>my-aws-account</code>. Wherever you see
<code>my-aws-account</code> appear below, please replace it with your
AWS account name.

1. Create VPC.
* Goto [Console](https://console.aws.amazon.com) > VPC.
* Click on **Start VPC Wizard**.
* On the **Step 1: Select a VPC Configuration** screen, make sure that
  **VPC with a Single Public Subnet** is highlighted and click
  **Select**.
* Name your VPC. Enter <code>defaultvpc</code> in the **VPC name** field.
* Enter <code>defaultvpc.internal.us-west-2</code> for **Subnet name**.
* Click **Create VPC**.

1. Create an EC2 role.
* Goto [Console](https://console.aws.amazon.com) > AWS Identity & Access Management > Roles.
* Click **Create New Role**.
* Set **Role Name** to <code>BaseIAMRole</code>. Click **Next Step**.
* On **Select Role Type** screen, hit **Select** for **Amazon EC2**.
* Click **Next Step**.
* On **Review** screen, click **Create Role**.
* EC2 instances launched with Spinnaker will be associated with this
role.

1. Create an EC2 Key Pair for connecting to your instances.
* Goto [Console](https://console.aws.amazon.com) > EC2 > Key Pairs.
* Click **Create Key Pair**.
* Name the key pair <code>my-aws-account-keypair</code>.
* AWS will download file <code>my-aws-account-keypair.pem</code> to
  your computer. <code>chmod 400</code> the file.

1. Create AWS credentials for Spinnaker.
* Goto [Console](https://console.aws.amazon.com) > AWS Identity & Access Management > Users > Create New Users. Enter a username and hit **Create**.
* Create an access key for the user. Click **Download Credentials**,
    then Save the access key and secret key into
    <code>~/.aws/credentials</code> in the spinnaker account of the
    AWS instance, as shown
    [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files).
* Click on the username you entered for a more detailed screen.
* On the **Summary** page, click on the **Permissions** tab.
* Click **Attach Policy**.
* Click the radio button next to **PowerUserAccess**, then
  click **Next Step**.
* Click on the **Inline Policies** header, then click the link to
  create an inline policy.
* Click **Select** for **Policy Generator**.
* Select **AWS Identity and Access Management** from the **AWS Service** pulldown.
* Select **PassRole** for **Actions**.
* Type <code>*</code> (the asterisk character) in the **Amazon Resource Name (ARN)** box.
* Click **Add Statement**, then **Next Step**.
* Click **Apply Policy**.

### Google Cloud Platform Setup

If you'd like to have Spinnaker deploy to and manage clusters on GCP,
you'll need to have a GCP project setup. If you've already got one,
please skip to the next step. Otherwise, please follow the
instructions below.

Sign into the [Google Developers
Console](https://console.developers.google.com) and create a
project. Call it a Project Name of <code>my-spinnaker-project</code> and
take note of the auto-generated Project ID (e.g. *powerful-surf-12345*).

1. Enable APIs in the <code>my-spinnaker-project</code> project.
  * Go to the API Management page.
  * Enable the [Compute
    Engine](https://console.developers.google.com/apis/api/compute_component/overview?project=_)
    and [Compute Engine
    Autoscaler](https://console.developers.google.com/apis/api/autoscaler/overview?project=_)
    APIs.

## Step 2: Set up a virtual machine to run Spinnaker

In this step, you'll set up a virtual machine to run Spinnaker on
either AWS or Google Cloud Platform. While you have the option of
selecting the machine type, we strongly recommend using a machine with
8 cores and at least 50GB RAM. If you'd like to run Spinnaker on your
local machine, feel free to skip this step and move on to [Step
3](#step-3-install-and-run-spinnaker).

### AWS Setup

Create an AWS virtual machine.

1. Goto [AWS Console](https://console.aws.amazon.com) > AWS Identity & Access
  Management > Roles.
* Click on **Create New Role**.
* Type "spinnakerRole" in the **Role Name** field. Hit **Next Step**.
* Click **Select** for the **Amazon EC2** service.
* Select the radio button next to **PowerUserAccess**, then click
  **Next Step**, followed by **Create Role**.
* Goto [AWS Console](https://console.aws.amazon.com) > EC2.
* Click **Launch Instance**.
* Click **Select** for the **Spinnaker-Ubuntu-14.04-2 - ami-04e1f065** image.
* Under **Step 2: Choose an Instance Type**, click the radio button
  for **m4.large**, then click **Next: Configure Instance Details**.
* Set the **Auto-assign Public IP** field to **Enable**, and the **IAM
  role** to "spinnakerRole".
* Click **Review and Launch**.
* Click **Launch**.
* Click **View Instances**. Make note of the **Public IP** field for
  the newly-created instance. This will be needed in the next step.

1. Shell in and open an SSH tunnel from your host to the virtual machine.
* Add this to ~/.ssh/config

          Host spinnaker
            HostName <Public IP address of instance you just created>
            IdentityFile </path/to/my-aws-account-keypair.pem>
            LocalForward 8081 127.0.0.1:9000
            LocalForward 8084 127.0.0.1:8084
            User ubuntu
* Execute

          ssh spinnaker

### Google Cloud Platform Setup

There are multiple ways to setup a virtual machine on Google Cloud
Platform for running Spinnaker. The instructions here do this by using
<code>gcloud</code>, Google Cloud Platform's command line interface
tool.

1. Install <code>gcloud</code>.
* If you already have <code>gcloud</code> installed, you may skip this
  step. Otherwise, please follow the [gcloud installation
  instructions](https://cloud.google.com/sdk).
1. Run

        gcloud auth login

1. To simplify your <code>gcloud</code> commands, it can be useful to set
   optional defaults. For example, if your Project ID is *powerful-surf-12345*,
   you can set this and your favorite zone with these two commands,

        gcloud config set project powerful-surf-12345
        gcloud config set compute/zone us-central1-f

1. Create a Google Compute Engine virtual machine. This command assumes you
   set the <code>gcloud</code> defaults above.

        gcloud compute instances create spinnaker-test --image ubuntu-14-04 --machine-type n1-standard-8 --scopes compute-rw

1. Shell in and open an SSH tunnel from your host to the virtual machine.

        gcloud compute ssh spinnaker-test --ssh-flag="-L 8084:localhost:8084" --ssh-flag="-L 9000:localhost:9000" --ssh-flag="-L 8087:localhost:8087"

## Step 3: Install and run Spinnaker

If you have chosen to run Spinnaker inside an Amazon Web Services or
Google Compute Engine VM, please ssh into the VM.

Irrespective of the machine on which you have chosen to run Spinnaker,
at the command prompt, type in (or cut-and-paste) the following
command:

    bash <(curl --silent https://spinnaker.bintray.com/scripts/InstallSpinnaker.sh)

The above installs and configures Spinnaker, and starts all Spinnaker
components, including Redis and Cassandra, which Spinnaker components
use to store data. If you see any errors, please just run the command
again.

Note that it can take several minutes for Spinnaker to start.

It will take several minutes to install and configure Spinnaker along with
all of its dependencies. Once the install is complete, you will use your
web browser to interact with Spinnaker.  If you are running Spinnaker on AWS,
point your browser at [http://localhost:8081](http://localhost:8081). Or, if
you are running Spinnaker on GCP, point your browser at
[http://localhost:9000](http://localhost:9000).

## Step 4. Configure example pipeline

To walk you through some of the basics with Spinnaker, you're going to
setup a Spinnaker pipeline that bakes a virtual machine (VM) image
containing redis, then deploys that image to a test cluster.

Note here that the us-east-1a availability zone is currently full, so it will need
to be deselected in the corresponding 'region' checkbox list for each of these
steps.

### Create a Spinnaker application

1. In Spinnaker, click **Actions** > **Create Application**
  1. Input <code>example</code> for the **Name** field and your email address for
the **Owner Email** field.
1. Click inside of the dashed rectangle beside the **Accounts** heading.
  * Click <code>my-aws-account</code> if you are deploying to AWS or
<code>my-google-account</code> if you are deploying to Google Cloud Platform.
1. Click on the **Consider only cloud provider health when executing
  tasks** button next to **Instance Health**.
1. Click the **Create** button.

### Create and configure a security group

Next, you'll create a security group that specifies traffic firewall
rules for the cluster. You'll configure the firewall rules to allow
all incoming traffic on port 80, for clusters associated with this
security group.

1. Click **SECURITY GROUPS**, then click the **+** button to create a security group.
1. Input <code>test</code> for the **Detail (optional)** field and
<code>Test environment</code> for the **Description** field.
1. If running on AWS, select **defaultvpc** as the **VPC** field and click **Next**.
1. If running on AWS
  * Click **Add new Security Group Rule**.
  * Click **default** on the **Security Group** dropdown.
  * Change **Start Port** and **End Port** to <code>80</code>.
1. If running on GCP
  * Click **Add New Source CIDR** and use the default
    <code>0.0.0.0/0</code> value for the **Source Range** field.
  * Click **Add New Protocol and Port Range**. Use the default
  <code>TCP</code> value for the **Protocol** field. Change **Start
  Port** and **End Port** to <code>80</code>.
1. Click the **Create** button.

### Create a load balancer

Next, you'll create a load balancer in Spinnaker.

1. Click **LOAD BALANCERS**, then click the **+** button to create a load balancer.
1. Input <code>test</code> for the **Stack** field.
1. If running on AWS, select **internal (defaultvpc)** from the **VPC
  Subnet** dropdown.
1. Click the **Next** button.
1. If running on AWS
  * Select **example-test** from the **Security Groups** dropdown.
  * Hit **Next**, then **Create**.
1. If running on GCP
  * Unselect the **Enable health check?** checkbox.
1. Click the **Create** button.

### Create a deployment pipeline

Your final task is to set up a Spinnaker pipeline. Let's name it
**Bake & Deploy to Test**. The pipeline will produce an image
containing the <code>redis-server</code> package and then deploy
it. In this tutorial, you'll trigger the pipeline manually.

To create the pipeline:

1. Click **PIPELINES**, then click **Configure** and select **Create
  New...** from the dropdown.
1. Input <code>Bake & Deploy to Test</code> for the **Pipeline Name**.
1. Click the **Create Pipeline** button.

#### Set up the first stage of the pipeline

You're now going to create the first stage of the pipeline. It will
build an image from an existing redis-server package.

1. Click **Add stage**.
1. Select **Bake** from the **Type** pulldown menu.
1. Input <code>redis-server</code> for the **Package** field.
1. Click **Save Changes**.

#### Set up the second stage of the pipeline

You're now going to setup the second stage of the pipeline. It takes
the image constructed in the *Bake* stage and deploys it into a test
environment.

1. Click **Add stage**.
1. Select **Deploy** from the **Type** dropdown.
1. Under the **Server Groups** heading, click **Add server group**.
1. Click the **Continue without a template** button.

1. Next, In the **Configure Deployment Cluster** window, input "test"
for the **Stack** field.

1. If running on AWS, select **defaultvpc** under **VPC Subnet**.
1. Click the **Next** button.
1. Click the text area next to the **Load Balancers** heading, then
  select <code>example-test</code>. Click the **Next** button.
1. Click the **Security Groups** form field, then click
  <code>example-test (example-test)</code>. Click the **Next**
  button.
1. If running on AWS
  * Click on the **Micro Utility** button to set the **Instance
    Profile**, then click **Next**.
  * Select the **Medium: m3** size, then click **Next**.
1. If running on GCP
  * Click on the **Micro Utility** button to set the **Instance
    Profile**, then click **Next**.
  * If running on GCP, select the **Micro** size, then click **Next**.
1. Input <code>2</code> for the **Number of Instances** field, then click the
  **Add** button.
1. Save the pipeline configuration by clicking the **Save Changes**
  button.

### Try it out!

1. Click **PIPELINES** in the navigation bar.
1. Click **Start Manual Execution** for the **Bake & Deploy to Test**
  pipeline.
1. Click **Run**.

Now, watch Spinnaker in action. A **MANUAL START** section will
appear, and will show progress as the pipeline executes. At any point
during pipeline execution, click on the horizontal bar to see detailed
status for any of the stages in the pipeline.

Feel free to navigate around the Spinnaker menus, create new
pipelines, clusters, server groups, load balancers, and security
groups, etc. and see what happens.

When you're ready to stop, don't forget to cleanup your resources. An
easy way to do this is to visit the pipelines, clusters, load
balancers, and security groups pages, click on the ones created and
select the appropriate **Delete** command from the Actions pulldown on
the right.

## Step 5. Stop Spinnaker

To stop Spinnaker, type:

    sudo stop spinnaker

The above command stops all Spinnaker components, including Redis and
Cassandra.
