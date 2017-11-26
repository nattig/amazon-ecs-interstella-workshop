# Interstella: CICD for Containers on AWS

## Overview
Welcome to the Interstella Galactic Trading Co team!  Interstella Galactic Trading Co is an intergalactic trading company that deals in rare resources.  Business is booming, but we're struggling to keep up with orders mainly due to our legacy logistics platform.  We heard about the benefits of containers, especially in the context of microservices and devops. We've already taken some steps in that direction, but can you help us take this to the next level? 

We've already moved to a microservice based model, but are still not able to develop quickly. We want to be able to deploy to our microservices as quickly as possible while maintaining a certain level of confidence that our code will work well. This is where you come in.

If you are not familiar with DevOps, there are multiple facets to the the word. One focuses on organizational values, such as small, well rounded agile teams focusing on owning a particular service, whereas one focuses on automating the software delivery process as much as possible to shorten the time between code check in and customers testing and providing feedback. This allows us to shorten the feedback loop and iterate based on customer requirements at a much quicker rate. 

In this workshop, you will take Interstella's existing logistics platform and apply concepts of CI/CD to their environment. To do this, you will create a pipeline to automate all deployments using AWS CodeCommit or GitHub, AWS CodeBuild, AWS CodePipeline, and AWS CloudFormation. Today, the Interstella logistic platform runs on Amazon EC2 Container Service following a microservice architecture, meaning that there are very strict API contracts that are in place. As part of the move to a more continuous delivery model, they would like to make sure these contracts are always maintained.

### Requirements:  
* AWS account - if you don't have one, it's easy and free to [create one](https://aws.amazon.com/)
* AWS IAM account with elevated privileges allowing you to interact with CloudFormation, IAM, EC2, ECS, ECR, ALB, VPC, SNS, CloudWatch, AWS CodeCommit, AWS CodeBuild, AWS CodePipeline
* A workstation or laptop with an ssh client installed, such as [putty](http://www.putty.org/) on Windows; or terminal or iterm on Mac
* Familiarity with Python, vim/emacs/nano, [Docker](https://www.docker.com/), AWS and microservices - not required but a bonus

### Labs:  
These labs are designed to be completed in sequence, and the full set of instructions are documented below.  Read and follow along to complete the labs.  If you're at a live AWS event, the workshop attendants will give you a high level run down of the labs and be around to answer any questions.  Don't worry if you get stuck, we provide hints and in some cases CloudFormation templates to catch you up.  

* **Workshop Setup:** Setup working environment on AWS  
* **Lab 1:** Containerize Interstella's logistics platform
* **Lab 2:** Deploy your container using ECR/ECS
* **Lab 3:** Break down the monolith to microservices and deploy using ECR/ECS
* **Bonus Lab:** Scale your microservices with ALB 


### Conventions:  
Throughout this workshop, we provide commands for you to run in the terminal.  These commands will look like this: 

<pre>
$ ssh -i <b><i>PRIVATE_KEY.PEM</i></b> ec2-user@<b><i>EC2_PUBLIC_DNS_NAME</i></b>
</pre>

The command starts after the $.  Text that is ***UPPER_ITALIC_BOLD*** indicates a value that is unique to your environment.  For example, the ***PRIVATE\_KEY.PEM*** refers to the private key of an SSH key pair that you've created, and the ***EC2\_PUBLIC\_DNS\_NAME*** is a value that is specific to an EC2 instance launched in your account.  You can find these unique values either in the CloudFormation outputs or by going to the specific service dashboard in the AWS management console. 

### Workshop Cleanup:
You will be deploying infrastructure on AWS which will have an associated cost.  Fortunately, this workshop should take no more than 2 hours to complete, so costs will be minimal.  When you're done with the workshop, follow these steps to make sure everything is cleaned up.  

1. Delete any manually created resources throughout the labs.  Certain things do not have a cost associated, and if you're not sure what has a cost, you can always look it up on our website.  All of our pricing is publicly available, or feel free to ask one of the workshop attendants when you're done. 
2. Delete any container images stored in ECR, delete CloudWatch logs groups, and delete ALBs (if you get to that lab) 
3. Delete the CloudFormation stack launched at the beginning of the workshop to clean up the rest.

* * * 

## Let's Begin!

### Workshop Setup

1\. Log into the AWS Management Console and select an [AWS region](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).  The region dropdown is in the upper right hand corner of the console to the left of the Support dropdown menu.  For this workshop, choose either **EU (Ireland)** or **EU (Frankfurt)**.  Workshop administrators will typically indicate which region you should use.

2\. Create an [SSH key pair](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) that will be used to login to launched EC2 instances.  If you already have an SSH key pair and have the PEM file (or PPK in the case of Windows Putty users), you can skip to the next step.  

Go to the EC2 Dashboard and click on **Key Pairs** in the left menu under Network & Security.  Click **Create Key Pair**, provide a name (e.g. interstella-workshop), and click **Create**.  Download the created .pem file, which is your private SSH key.      

*Mac or Linux Users*:  Change the permissions of the .pem file to be less open using this command:

<pre>$ chmod 400 <b><i>PRIVATE_KEY.PEM</i></b></pre>

*Windows Users*: Convert the .pem file to .ppk format to use with Putty.  Here is a link to instructions for the file conversion - [Connecting to Your Linux Instance from Windows Using PuTTY](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html)

3\. Generate a Fulfillment API Key for the logistics software [here](http://www.interstella.trade/getkey.html).  Create a username and password to login to the API Key Management portal; you'll need to access this page again later in the workshop, so don't forget what they are.  Click **GetKey** to generate an API Key.  Note down your username and API Key because we'll be tracking resource fulfillment rates.  The API key will be used later to authorize the logistics software send messages to the order fulfillment API endpoint (see arch diagram in Lab 1).

4\. For your convenience, we provide a [CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template to stand up core workshop infrastructure.

Here is what the workshop environment looks like:

![CloudFormation Starting Stack](images/arch-starthere.png)

The CloudFormation template will launch the following:
* VPC with public subnets, routes and Internet Gateway
* EC2 Instances with security groups (inbound tcp 22, 80, 5000) and joined to an ECS cluster 
* ECR repositories for your container images
* Application Load Balancer to front all your services
* Parameter store to hold values for your API Key, Order Fulfillment URL, SNS Topic ARNs to subscribe to, and a security SNS topic.

*Note: SNS Orders topic, S3 assets, API Gateway and DynamoDB tables are admin components that run in the workshop administrator's account.  If you're at a live AWS event, this will be provided by the workshop facilitators.  We're working on packaging up the admin components in an admin CloudFormation template, so you can run this workshop at your office, home, etc.*

Click on the CloudFormation launch template link below for the region you selected in Step 1.  The link will load the CloudFormation Dashboard and start the stack creation process in the specified region.

Region | Launch Template
------------ | -------------  
**Ireland** (eu-west-1) | [![Launch Interstella Stack into Ireland with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/new?stackName=amazon-ecs-interstella-workshop-3&templateURL=https://s3-us-west-2.amazonaws.com/www.interstella.trade/workshop3/starthere.yaml)  
**Frankfurt** (eu-central-1) | [![Launch Interstella Stack into Frankfurt with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/new?stackName=amazon-ecs-interstella-workshop-3&templateURL=https://s3-us-west-2.amazonaws.com/www.interstella.trade/workshop3/starthere.yaml)

The link above will bring you to the AWS CloudFormation console with the **Specify an Amazon S3 template URL** field populated and radio button selected. Just click **Next**. If you do not have this populated, please click the link above.

![CloudFormation Starting Stack](images/cfn-createstack-1.png)

In the **Specify Details** page, there are some parameters to populate. Feel free to leave any of the pre-populated fields as is. The only fields you NEED to change are:

- EnvironmentName - *This name will be prepended to many of the resources created to help you distinguish the workshop resources from other existing ones*
- InterstellaApiKey - *In a previous step, you visited the getkey website to get an API key for fulfillment. Enter it here.*
- KeyPairName - *You will need to log into an EC2 instance. This is your authentication mechanism. If there are no options in the dropdown, please create a new keypair*

Click **Next**

In the **Options** section, you can leave things blank and default. You can optionally enter in tags to be applied to all resources.

In the **Review** section, take a look at all the parameters and make sure they're accurate. Check the box next to **I acknowledge that AWS CloudFormation might create IAM resources with custom names.** as the CloudFormation template will create IAM roles on your behalf for this workshop. As part of the cleanup, CloudFormation will remove the IAM Roles for you.



### Lab 1 - Offload the application build from your dev machine

In this lab, you will start the process of automating the entire software delivery process. The first step we're going to take is to automate the Docker container builds and push the container image into the EC2 Container Registry. This will allow you to develop and not have to worry too much about build resources. We will use AWS CodeCommit and AWS CodeBuild to automate this process. 

We've already separated the code in the application, but it's time to move the code out of the monolith repo so we can work on it quicker. As part of the bootstrap process, CloudFormation has already created an AWS CodeCommit repository for you. It will be called {EnvironmentName}-iridium. We'll use this repository to break apart the iridium microservice code from the monolith. 

1\. Connect to your AWS CodeCommit repository.

In the AWS Management Console, navigate to the AWS CodeCommit dashboard. Choose the repository named **Environmentname-iridium-repo** where Environmentname is what you entered in CloudFormation. A screen should appear saying **Connect to your repository**.
*Note: If you are familiar with using git, feel free to use the ssh connection so you don't have to type in a password every time.*

When the **Connect to your repository** menu appears, choose **HTTPS** for the connection type to make things simpler for this lab. Then follow the **Steps to clone your repository**. Click on the **IAM User** link. This will generate credentials for you to log into CodeCommit when trying to check your code in. 

![CodeCommit Connect Repo](3-4-codecommitcreateiam.png)

Scroll down to the **HTTPS Git credentials for AWS CodeCommit** section and click on **Generate**. 

![Codecommit HTTPS Credentials](3-5-codecommitgeneratecreds.png)

Save the **User name** and **Password** as you'll never be able to get this again.

2\. Create and configure an AWS CodeBuild Project.

*You may be thinking, why would I want this to automate when I could just do it on my local machine. Well, this is going to be part of your full production pipeline. We'll use the same build system process as you will for production deployments. In the event that something is different on your local machine as it is within the full dev/prod pipeline, this will catch the issue earlier. You can read more about this by looking into **Shift Left**.*

In the AWS Management Console navigate to the AWS CodeBuild console. If this is your first time using AWS CodeBuild in your region, you will have to click **Get Started**. Otherwise, create a new project.

On the **Configure your project** page, enter in the following details:

Project Name: **dev-iridium-service**

Source Provider: **AWS CodeCommit**

Repository: **Your AWS CodeCommit repository name from above**

Type: **No Artifacts**

Service Role: **Create a service role in your account**

Role Name: **codebuild-dev-iridium-service-role**

![CodeBuild Create Project](png)

Expand the **Advanced** section.

Under Environment Variables, enter two variables:

Name: **AWS_ACCOUNT_ID** Value: **Your account ID** Type: **Plaintext**

Name: **IMAGE_REPO_NAME** Value: **YOURENVIRONMENTNAME-iridium** Type: **Plaintext**

Click **Continue**

Click **Save**

When you click save, CodeBuild will create an IAM role to access other AWS resources to complete your build. By default, it doesn't include everything, so we will need to modify the newly created IAM Role to allow access to EC2 Container Registry (ECR). 

3\. Modify IAM role to allow CodeBuild to access other resources like ECR.

In the AWS Management Console, navigate to the AWS IAM console. Choose **Roles** on the left. Find the role that created earlier. In the example, the name of the role created was **codebuild-dev-iridium-service-role**. Click **Add inline policy**. By adding an inline policy, we can keep the existing managed policy separate from what we want to manage ourselves. 

Choose **Custom Policy**. Name it **AccessECR** and enter in:

<pre>
`{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:CompleteLayerUpload",
                "ecr:GetAuthorizationToken",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:UploadLayerPart"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}`

</pre>

Choose **Apply Policy**

4\. Get details on EC2 Container Repository where we will be pushing and pulling Docker images to/from.

We now have the building blocks in place to start automating the builds of our Docker images. Now it's time to figure out how to use the Amazon EC2 Container Registry. 

In the AWS Management Console, navigate to the Amazon EC2 Container Service console. On the left, click on Repositories. You'll see a number of repositories that have already been created for you. Choose the one for melange and click on *View Push Commands* to get the commands to login, tag, and push to this particular ECR repo.

Copy down these instructions somewhere as we will need to add them into our code later. Alternatively, bookmark this page so you can come back.

![ECR Instructions](png)

5\. SSH into one of the launched EC2 instances to get working with the code. 

Go to the EC2 Dashboard in the Management Console and click on **Instances** in the left menu.  Select either one of the EC2 instances created by the CloudFormation stack and SSH into the instance.  

*Tip: If your instances list is cluttered with other instances, type the **EnvironmentName** you used when running your CloudFormation template into the filter search bar to reveal only those instances.*  

![EC2 Public IP](images/1-ec2-IP.png)

<pre>
$ ssh -i <b><i>PRIVATE_KEY.PEM</i></b> ec2-user@<b><i>EC2_PUBLIC_IP_ADDRESS</i></b>
</pre>

If you see something similar to the following message (host IP address and fingerprint will be different, this is just an example) when trying to initiate an SSH connection, this is normal when trying to SSH to a server for the first time.  The SSH client just does not recognize the host and is asking for confirmation.  Just type **yes** and hit **enter** to continue:

<pre>
The authenticity of host '52.15.243.19 (52.15.243.19)' can't be established.
RSA key fingerprint is 02:f9:74:ef:d8:5c:19:b3:27:37:57:4f:da:37:2b:e8.
Are you sure you want to continue connecting (yes/no)? 
</pre>

6\. Clone workshop repo and commit one microservice to your repo

Once logged in, clone your new repository and the workshop repository. Go back to the AWS CodeCommit console, click on your repository, and then copy the command to clone your empty repository.

**Make sure to replace YOURENVIRONMENTNAME with the name you put into CloudFormation for the following commands**

<pre>
$ git clone https://git-codecommit.*your_region*.amazonaws.com/v1/repos/*YOURENVIRONMENTNAME-iridium*
$ git clone https://git-codecommit.*your_region*.amazonaws.com/v1/repos/*YOURENVIRONMENTNAME-monolith*
$ git clone https://github.com/interstellaworkshop
</pre>

Now that we have everything locally, we can move just one microservice into the new repo. Because this is for development, we'll be checking into development. 

*You are now separating one part of the repository into another so that you can commit direct to the specific service. Similar to breaking up the monolith application in lab 2, we've now started to break the monolithic repository apart.*



<pre>
$ cp -R ecs-interstella-workshop/lab3/iridium/ YOURENVIRONMENTNAME-iridium/
$ cd YOURENVIRONMENTNAME-iridium
$ git checkout -b dev
$ git add -A
$ git commit -m "Splitting iridium service into its own repo"
$ git push origin dev
</pre>

If we go back to the AWS CodeCommit dashboard, we should now be able to look at our code we just committed.

![CodeCommit Code Committed](png)

7\. Add a file to instruct AWS CodeBuild on what to do.

AWS CodeBuild uses a definition file called a buildspec.yml file. The contents of the buildspec will determine what AWS actions CodeBuild should perform. The key parts of the buildspec are Environment Variables, Phases, and Artifacts. 

At Interstella, we want to follow best practices, so there are 2 requirements:
1. We don't use the *latest* tag for Docker images. We have decided to use the Commit ID from our source control instead as the tag so we know exactly what image was deployed.
2. We want this buildspec to be generalized to multiple environments but use different CodeBuild projects to build our Docker containers. You have to figure out how to do this

Again, one of your lackeys has started a buildspec file for you, but never got to finishing it. Add the remaining instructions to the buildspec.yml.draft. The file should bein your YOURENVIRONMENTNAME-iridium folder and already checked in. Copy the draft to a buildspec.yml file.

<pre>
$ cp buildspec.yml.draft buildspec.yml
</pre>

Now that you have a copy of the draft as your buildspec, you can start editing it. If you get stuck, look at the hintspec.yml file also in the same folder.

Here are some links to the documentation and hints:

Log into ECR: http://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html
*disregard the --no-include-email portion*
Building a Docker image: https://docs.docker.com/get-started/part2/#build-the-app
Available Environment Variables in CodeBuild:  http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html

8\. Check in your new file into the AWS CodeCommit repository.
<pre>
$ git add buildspec.yml
$ git commit -m "Adding in support for AWS CodeBuild"
$ git push origin dev
</pre>

9\. Test your build.

In the AWS CodeBuild console, choose the melange project and build it. Select the **dev** branch and the newest commit should populate automatically. Your commit message should appear as well. Then click **Start Build**. 

If all goes well, you should see a lot of successes and your image in the ECR console. Inspect the **Build Log** if there were any failures. You'll also see these same logs in the CloudWatch Logs console. 

![Successes in CodeBuild](png)

10\. Link AWS CodeCommit and AWS CodeBuild to automate a build of your dev branch.
**Can I do pull request and build?**

### Lab 2 - Automate end to end deployments

In this lab, you will build upon the process that you've already started by introducing an orchestration layer to control builds, deployments, and more. To do this, you will create a pipeline to automate all deployments using AWS CodeCommit or GitHub, AWS CodeBuild, AWS CodePipeline, and AWS CloudFormation. Today, the Interstella logistic platform runs on Amazon EC2 Container Service following a microservice architecture, so we will be deploying to an individual service. 

There are a few changes we'll need to make to the code that we used as part of Lab 1. For example, the buildspec we used had no artifacts, but we will need artifacts to pass variables on to the next stage. 

1\. Create a master branch and check in your new code including the buildspec.

Log back into your EC2 instance and create a new branch in your CodeCommit repository.

<pre>
$ cd YOURENVIRONMENTNAME-iridium
$ git checkout -b master
$ git merge dev
$ git push origin master
</pre>

2.\ Take a look at the AWS CloudFormation template named service.yml. 





<link here>




Since we're starting fresh, it's best to try and control everything using CloudFormation. Looking at this template that has already been created, it's generalized to take a cluster, desired count, tag, target group, and repository. This means that you'll have to pass the variables to CloudFormation to create the stack. CloudFormation will take the parameters and create an ECS service that matches the parameters.

3\. Update AWS CodeBuild buildspec.yml to support deployments to AWS CloudFormation

While using AWS CodePipeline, we will need a way of passing variables to different stages. The buildspec you created earlier had no artifacts, but now there will be two artifacts. One for the AWS CloudFormation template and one will be for parameters to pass to AWS CloudFormation when launching the stack. As part of the build, you'll also need to create the parameters to send to AWS CloudFormation and output them in JSON format. 

As part of the initial bootstrapping, we've already created a target group for you for the Application Load Balancer. The name of the target group can be accessed from the EC2 Parameter Store under **interstella/iridium-target-group**. You'll need to pull in those parameters as part of the buildspec. How will you do it?

Open the existing **buildspec.yml** for the **iridium** microservice. 

*Add a section for parameters and pull in the right parameters from parameter store. Specifically, we'll need to get the parameters iridiumTargetGroupArn, cloudWatchLogsGroup, and ecsClusterName so we can pass those to the CloudFormation stack later.

Add a line to put all the parameters into JSON format and write it to disk as build.json. The parameters in this build.json file should map 1:1 to the parameters in service.yml

Add a section for artifacts and include the build.json file and also the service.yml CloudFormation template.*

<details>
  <summary>
    ***Click here to show detailed info on what to do in the buildspec file***
  </summary>

Here are some hints and documentation:

[Buildspec reference](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html) 

Add a section to your buildspec.yml file entitled "**env**". Within this section you can either choose regular environment variables, or pull them from parameter store, which is what we will do. It should look something like this: 

<pre>
env:
  parameter-store:
    targetGroup: /interstella/iridiumTargetGroupArn
    ...
</pre>

Now, look at the service.yml file. We need to have CodeBuild output all the parameters so CloudFormation can take them in as inputs. The parameters in service.yml are: 

<pre>
Tag:
  Type: String

DesiredCount:
  Type: Number

TargetGroupArn:
  Type: String

Cluster:
  Type: String

Repository:
  Type: String
  
cloudWatchLogsGroup:
  Type: String

CwlPrefix:
  Type: String
</pre>

Now let's write a parameters file that will map my parameters to that of the CloudFormation template. 

<pre>
...
commands:
  ...
  - printf '{"Parameters":{"Tag":"%s","DesiredCount":"2","TargetGroupArn":"%s","Cluster":"%s","Repository":"%s", "cloudWatchLogsGroup":"%s","CwlPrefix":"%s"}}' $TAG $targetGroup $ecsClusterName $IMAGE_REPO_NAME $cloudWatchLogsGroup $ENV_TYPE > build.json
</pre>

If you get stuck, look at the file **finalhelpspec.yml** 

</details>
4\. Create an AWS CodePipeline Pipeline and set it up to listen to AWS CodeCommit. 

In the AWS Management Console, navigate to the AWS CodePipeline console. Click on **Create Pipeline**.

*Note: If this is your first time visiting the AWS CodePipeline console in the region, you will need to click "**Get Started**"*

We're going to make this a production pipeline. Name the pipeline "**prod-iridium-service**". Click Next.

![CodePipeline Name](3-6-codepipelinename.png)

For the Source Location, choose **AWS CodeCommit**. Then, choose the repository you created as in Step 1. 

*Here, we are choosing what we want AWS CodePipeline to monitor. Using Amazon CloudWatch Events, AWS CodeCommit will trigger when something is pushed to a repo.*

![CodePipeline Source](3-7-codepipelinesource.png)

Next, configure the Build action. Choose **AWS CodeBuild** as the build provider. Click **Create a new build project** and name it **prod-iridium-service**.

![CodePipeline Build](3-8-codepipelinebuildproject.png)

Scroll down further. In the Environment: How to build section, select values for the following fields:

- **Environment Image: Use an Image managed by AWS CodeBuild** - *There are two options. You can either use a predefined Docker container that is curated by CodeBuild, or you can upload your own if you want to customize dependencies etc. to speed up build time*
- **Operating System: Ubuntu** - *This is the OS that will run your build*
- **Runtime: Docker** - *Each image has specific versions of software installed. See [Docker Images Provided by AWS CodeBuild](http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html)*
- **Version: aws/codebuild/docker:1.12.1** - *There's only one version now, but you will be able to choose different versions in the future*

Expand the **Advanced** section.

Under Environment Variables, enter three variables:

- Name: **AWS_ACCOUNT_ID** Value: **Your account ID** Type: **Plaintext** *Earlier, when we created the buildspec, it looked for some existing environment variables like this one.*
- Name: **IMAGE_REPO_NAME** Value: **YOURENVIRONMENTNAME-iridium** Type: **Plaintext** 
- Name: **ENV_TYPE** Value: **prod** Type: **Plaintext** *This is a new environment variable which we're going to use to prefix our log stream. You'll see in lab 3 why this is needed*

Once confirmed, ensure **Create a service role in your account** is selected and leave the name as default. When you're done, choose **Save build project**. AWS CodeBuild will create a project for you which should take about 10 seconds. Once it's done, click **Next Step**.

![CodePipeline Build Pt2](3-9-codepipelinebuildprojectpt2.png])

The next dialog that will appear is **Deploy**. Select and populate the following values:

- **Deployment provider: AWS CloudFormation** - *This is the mechanism we're choosing to deploy with. CodePipeline also supports several other deployment options, but we're using CloudFormation in this case.*
- **Action Mode: Create or Update a Stack** - *This will actually update the stack instead of just creating a change set. We'll do that later.*
- **Stack Name: prod-iridium-service** - *Name the CloudFormation stack that you're going to create/update*
- **Template File: service.yml** - *The filename of the template that you looked over earlier in this workshop*
- **Configuration File: build.json** - *The filename of the JSON file generated by CodeBuild that has all the parameters*
- **Capabilities: CAPABILITY_IAM** - *Here, we're giving CloudFormation the ability to create IAM resources*
- **Role Name: The name of the role created as part of the initial CloudFormation stack** - *CloudFormation needs a role to assume so that it can create and update stacks on your behalf*

![CodePipeline Deploy](3-10-codepipelinedeploy.png)
*Todo: CodePipeline Deploy to ECS direct.*

In the next step of creating your pipeline, we must give AWS CodePipeline a way to access artifacts and dependencies to pull. Leave the Role Name blank and click **Create Role**. 

![CodePipeline Role Creation](3-11-codepipelinerolecreation.png)

You will be automatically taken to the IAM Management Console to create a service role. Choose **Create a new IAM Role** and leave the role name as the default. Click **Allow** to create the role and return to AWS CodePipeline. Click **Next Step**

![CodePipeline Role IAM](3-12-codepipelineroleiam.png)

When you return to the AWS CodePipeline Console, click in the blank dialog box for Role Name and choose the newly created IAM Role. Click **Next Step**

![CodePipeline Role Select](3-13-codepipelineroleselect.png)

Review your pipeline and click **Create pipeline**.

5\. Test your pipeline.

Once the pipeline is created, CodePipeline will automatically try to get the most recent commit from the source. In this case, that's CodeCommit. Navigate to the AWS CodePipeline Console and choose your **prod-iridium-service** pipeline. You should see it working.

**Oh no!** There seems to have been an error! Let's try and troubleshoot it. Try and click around to find out what's wrong. Expand the section below to get some help.

<details>
  <summary>
    *Click here to expand this section and we'll go over how to find out what happened.*
  </summary>
  From the pipeline, it's easy to see that the whole process failed at the build step. Let's click on **Details** to see what it will tell us.

  Now click on **Link to execution details** since the error message didn't tell us much.

  The link brings you to the execution details of your specific build. We can look through the logs and the different steps to find out what's wrong. In this case, it looks like the **PRE_BUILD** step failed with the output message of **Error while executing command: $(aws ecr get-login --region $AWS_DEFAULT_REGION). Reason: exit status 255**

  Looking in the logs, we can see that **AccessDeniedException: User: arn:aws:sts::123456789012:assumed-role/code-build-prod-iridium-service-service-role/AWSCodeBuild-e111c11e-b111-11c1-ac11-f1111a1f1c11 is not authorized to perform: ssm:GetParameters on resource: arn:aws:ssm:us-east-2:123456789012:parameter/interstella/orderTopic status code: 400**

  Right, we forgot to give AWS CodeBuild the permissions to do everything it needs to do. Copy the region and account number as we'll be using those. Let's go fix it. 

  In the AWS Management Console, navigate to the AWS IAM console. Choose **Roles** on the left. Find the role that created earlier. In the example, the name of the role created was **code-build-prod-iridium-service-service-role**. Click **Add inline policy**. By adding an inline policy, we can keep the existing managed policy separate from what we want to manage ourselves. 

  Choose **Custom Policy**. Name it **AccessECR**. In the Resource section for ssm:GetParameters, make sure you replace the REGION and ACCOUNTNUMBER so we can lock down CodeBuild's role to only access the right parameters. Enter the following policy:

<pre>
`{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:CompleteLayerUpload",
                "ecr:GetAuthorizationToken",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:UploadLayerPart"
            ],
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": [
              "ssm:GetParameters"
            ],
            "Resource": "arn:aws:ssm:REGION:ACCOUNTNUMBER:parameter/interstella/*",
            "Effect": "Allow"
        }
    ]
}`

</pre>

Choose **Apply Policy**

</details>

Once you think you've fixed the problem, since the code and pipeline haven't actually changed, we can retry the build step. Navigate back to the CodePipeline Console and choose your pipeline. Then click the **Retry** button in the Build stage.

### Checkpoint:  
At this point you have a pipeline ready to listen for changes to your repo. Once a change is checked in to your repo, CodePipeline will bring your artifact to CodeBuild to build the container and check into ECR. AWS CodePipeline will then deploy your application into ECS using the existing task definitions. Check in a change and you'll see the pipeline kick off.




### Lab 3 - Add Security and Implement Automated Testing
Now that we've automated the deployments of our application, we want to improve the security posture of our application, so we will be automating the testing of the application as well. We've decided that as a starting point, all Interstella deployments will require a minimum of 1 test to make the changes minimal for developers. At the same time,  we will start using [IAM Roles for Tasks](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html). This allows us to specify a role per task instead of assuming the EC2 Instance Role. 

In this lab, we will use Stelligent's tool **cfn-nag**. The cfn-nag tool looks for patterns in CloudFormation templates that may indicate insecure infrastructure. Roughly speaking it will look for:

- IAM rules that are too permissive (wildcards)
- Security group rules that are too permissive (wildcards)
- Access logs that aren't enabled
- Encryption that isn't enabled

For more background on the tool, please see: [Finding Security Problems Early in the Development Process of a CloudFormation Template with "cfn-nag"](https://stelligent.com/2016/04/07/finding-security-problems-early-in-the-development-process-of-a-cloudformation-template-with-cfn-nag/)

1\. Add in IAM task roles in service.yml

Now that the microservices are really split up, we should look into how to lock them down. One great way is to use IAM Roles for Tasks. We can give a specific task an IAM role so we know exactly what task assumed what role to do something instead of relying on the default EC2 instance profile.

A complete and updated service.yml file is located in hints/final-service.yml. Overwrite your existing service.yml with that one. 

Here are the differences:

- We added a new role to the template:

<pre>
ECSTaskRole:
  Type: AWS::IAM::Role
  Properties:
    Path: /
    AssumeRolePolicyDocument: |
      {
          "Statement": [{
              "Effect": "Allow",
              "Principal": { "Service": [ "ecs-tasks.amazonaws.com" ]},
              "Action": [ "sts:AssumeRole" ]
          }]
      }
    Policies: 
      - 
        PolicyName: "root"
        PolicyDocument: 
          Version: "2012-10-17"
          Statement: 
            - 
              Effect: "Allow"
              Action: "*"
              Resource: "*"
</pre>

- Then updated the ECS task definition to use the new role.

<pre>
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    Family: iridium
    ContainerDefinitions:
      - Name: iridium
        Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${Repository}:${Tag}
        Essential: true
        Memory: 128
        PortMappings:
          - ContainerPort: 5000
        Environment:
          - Name: Tag
            Value: !Ref Tag
        <b>TaskRoleArn: !Ref ECSTaskRole </b>
        LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group:
                Ref: cloudWatchLogsGroup
              awslogs-region:
                Ref: AWS::Region
              awslogs-stream-prefix: prod
</pre>

Create a new stage for testing in your pipeline

Navigate to the AWS CodePipeline dashboard and choose your pipeline. Edit the pipeline and click the **+stage** button between source and the build stage. Name it **CodeAnalysis** and then click on the **+ Action** button. This will add a new stage to your pipeline where we can run some static analysis. *We want this static analysis tool to run before our Docker container even gets built so that we can fail the deployment quickly if something goes wrong.*

Select and populate the following Values:

- **Action Category - Test**
- **Action Name - StaticAnalysis**
- **Test provider - AWS CodeBuild**
- **Project Name - CFN Value** - *We've already created a CodeBuild project for you as part of the initial CloudFormation stack. It's a Ruby stack as cfn-nag uses ruby.*
- **Input Artifact #1 - MyApp**

Click **Add Action**

2\. Create a new yml file for the test CodeBuild project to use.

In the CloudFormation stack, we configured the CodeBuild project to look for a file named **test-buildspec.yml**. With this, CodeBuild will install cfn-nag and then scan the service.yml CloudFormation template. It's the same format as buildspec.yml you used earlier. Take a look at the [Stelligent cfn-nag github repo](https://github.com/stelligent/cfn_nag) for how to install it. We've placed a test-buildspec.yml.draft in the service folder for you to start. It looks like this:

<pre>
version: 0.2

phases:
  pre_build:
    commands:
      - gem install cfn-nag
  build:
    commands:
      - echo 'In Build'
      - cfn_nag_scan --input-path service.yml
</pre>

<details>
  <summary>
    Click here for some assistance.
  </summary>
  Within the pre-build stage, we'll want to install cfn-nag. Then we'll want to use the cfn_nag_scan command to scan the service.yml CloudFormation template. It should look like this:
  <pre>
  version: 0.2

  phases:
    pre_build:
      commands:
        - gem install cfn-nag
    build:
      commands:
        - echo 'In Build'
        - cfn_nag_scan --input-path service.yml
  </pre>
  
  A completed file is in the hints folder of workshop3. It's named hint1-test-buildspec.yml
</details>

You should be good to go for the cfn-nag build now, but why stop here? Let's add in a few more lines to look for any sort of AWS Access or Secret keys. Can you think of a way to do this? AWS Access keys (as of the writing of this workshop) are alphanumeric and 20 characters long. Secret keys, however, can contain some special characters and are 40 characters long. How would you look through your code for anything like this and throw a warning up if something exists?

<details>
<summary>
  Click here for an answer that we've come up with.
</summary>
  We've pre-written a script for you to look for an AWS Access Key or Secret Key within your code. Take a look in github for the [checkaccesskeys.sh script in GitHub](https://github.com/aws-samples/amazon-ecs-interstella-workshop/blob/master/workshop3/tests/checkaccesskeys.sh). If it finds something, it will output some warnings to the CodeBuild log output. Normally, we would fire off some sort of security notification, but this will do for now. 

  Within the build section, add in a line to run a script in the test folder.
  <pre>
  ./tests/checkaccesskeys.sh 
  </pre>

  Your test-build.yml should now look like this:

  <pre>
  version: 0.2

  phases:
    pre_build:
      commands:
        - gem install cfn-nag
    build:
      commands:
        - echo 'In Build'
        - cfn_nag_scan --input-path service.yml
        - ./tests/checkaccesskeys.sh
  </pre>

  A final version of this test-buildspec.yml is also located in the hints folder. It's named final-test-buildspec.yml.
</details>

Let's check everything in and run the test. 

<pre>
$ git add test-cfn-nag.yml
$ git commit -m "Adding in buildspec for cfn-nag"
$ git push origin master
</pre>

3\. 






















### Lab 3 - Implement Canary Deployments
Now that we've automated deployments, we want to make sure our new code works before deploying it out to our fleet. It's important to verify that deployments actually worked before deploying them out across the board. To do this, we ill use the concept of Canary deployments. 

Canary deployments allow you to introduce a new container into your service without deploying everything. In ECS, we will run a service with just one container but add it to the same target group as the original service. We will monitor via metrics and logs. Specifically, we're going to look for errors in CloudWatch Logs under the prefix **canary**.

We'll use CodePipeline to create a gate before deploying to production. A gate allows you to manually review the output and the deployments before continuing. Then we'll add another stage to deploy one service just before the gate.

Let's do this!

1\. First, let's subscribe yourself to the precreated SNS endpoint for the approvals. 

Navigate to the AWS CloudFormation dashboard in the AWS Management console. Navigate to the AWS SNS dashboard in the AWS Management Console.  Choose **Topics** on the left hand side. Click the topic that was created as part of the CloudFormation stack. It will be named the value you entered for EnvironmentName.

Choose **Create Subscription**. Select **Email** as the protocol and enter in your email as the endpoint. 

Click **Create Subscription**

You'll soon get an email from SNS asking to confirm the subscription. Confirm it.

2\. Add a gate to your pipeline.

Navigate to the AWS CodePipeline dashboard and choose your pipeline. Edit the pipeline and click the **+stage** button between build and the deploy stage. Then click on the **+ Action** button. 

Select the following options in the **Add Action** dialog:

Action Category: **Approval**

Action Name: **ApproveProdDeployment**

Approval Type: **Manual Approval**

SNS Approval ARN: **The Topic You Subscribed to**

Click **Update**

Enter a **Stage Name** of **ApproveProdDeployment**

Click **Save pipeline changes** and then **Save and Continue**

3\. Add another stage, this time make it a deployment stage

Click **Edit** and add a stage right before **ApproveProdDeployment**. Name it **CanaryProdDeploy**.

Enter the following values: 

**Action category: Deploy**

**Action name: CanaryProdDeploy**

**Deployment provider: AWS CloudFormation**

**Action Mode: Create or Update a Stack**

**Stack name: canary-iridium-service**

**Template: MyAppBuild::service.yml**

**Template configuration: MyAppBuild::build.json**

**Capabilities: CAPABILITY_IAM**

**Role Name: The role you entered in previously**

**Output File name: Blank**

**Parameter Overrides:**
<pre>
{
  "DesiredCount": "1",
  "CwlPrefix": "canary"
} 
</pre>

**Input Artifacts #1: MyAppBuild**

Choose **Add Action** and save the pipeline.

3\. Test the pipeline

Click **Release Change** 

1\. Create another stage for testing in your existing pipeline

Navigate to the AWS CodePipeline dashboard and choose your pipeline. Edit the pipeline and add in a Test stage. 





Here is a reference architecture for what you will be implementing in Lab 1:
![Lab 1 Architecture](lab1.png)

6\. Now that we have a branch to test, let's make sure it builds locally. In this case, we're creating a Docker image. 

<pre>
$ docker build -t melange-microservice:test .
$ docker run -it 
$ log into ECR and push
</pre>

This should give you output similar to: 

CodeCommit: