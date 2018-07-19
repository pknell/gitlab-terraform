# Terraform Deployment with GitLab & AWS
*© 2018 Paul Knell, NVISIA LLC*

[Terraform](https://www.terraform.io) is a useful tool for scripting the deployment of resources into
Amazon's Cloud (AWS).  Usually one begins using Terrform at the local command
line--by writing configuration files and then running them with "terraform apply"
commands. It doesn't take long, though, before one realizes that the local
backend (the "terraform.tfstate" file) isn't enough for collaborative projects
where deployment pipelines might run concurrently, therefore the [S3 Backend](https://www.terraform.io/docs/backends/types/s3.html)
should be used. The use of S3 to store the Terraform state is simple when there's
just one environment, but when you need to support multiple (e.g., test, staging,
production) there's a bit more to it... you can either:
1. Use a single S3 Bucket with different folders for the various environments, using [Terraform's workspace feature](https://www.terraform.io/guides/running-terraform-in-automation.html#multi-environment-deployment).
1. Or, you could use a separate S3 Bucket for each environment.

We like to use different AWS accounts for each environment, particularly because it isolates them and helps distinguish costs on the monthy bill.
For this article, we'll use the second approach--each account (i.e., each environment) gets it's own "Terraform State" S3 Bucket.

For the CI/CD server, however, there's only one for all environments--to keep costs down.
This blog works through the creation of this kind of DevOps architecture (as depicted below), and provides a couple CloudFormation templates to make implementation easier.

![Figure 1](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/figure1.png)

A few things to note about the above diagram:
* The GitLab Server can either be self-managed by deploying your own instance into the DevOps Account (right next to the GitLab Runner), or the GitLab Runner can be configured to point at "gitlab.com". It's more leg-work to run your own, but there are advantages.
* The updates that Terraform performs against each environment can be either: automatic, periodically scheduled, or push-button (i.e., manual review/approval)--it all depends on how you configure the GitLab Pipeline.
* Only "Staging" and "Production" are depicted, but it's easy to add more environments.

## Getting Started

You're going to need a GitLab account, and at least two AWS accounts (or three if you want to set up multiple environments, such as "staging" and "production").

In the GitLab account, create a new blank project:

![Create GitLab Project](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/create-gitlab-project.png)

This will be the project for the CI/CD Pipeline.
Initially, you can create it with just a README.md file, but later we'll add a ".gitlab-ci.yaml" and a "main.tf" file.

Go to the project's CI/CD settings, and click the "Disable Shared Runners" button because we'll be using our own runner:
![Disable Shared Runners](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/disable-shared-runners.png)

In each AWS account, if you're following the recommended practice of using an IAM user (rather than the root account credentials), make
sure your user has console access and sufficient permissions to create the CloudFormation stack. The DevOps account's user
will need administrative access to: IAM, Lambda, and EC2. The "Staging" and "Production" account's user will need
administrative access to: IAM, S3, DynamoDB, and EC2.

## Deploy the GitLab Runner

Log-in to the AWS console using your DevOps account, and navigate into CloudFormation (using Services --> CloudFormation).

Click "Create Stack".

Specify the template by uploading the gitlab-runner.template file:

![Upload Template to S3](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/upload-template.png)

Click "Next", and then enter each missing parameter:
* Stack name: the stack name, such as "GitLab-Runner"
* GitLabApiToken: For now, leave this blank. It's used for clean-up of the runner's registration, which will be discussed later.
* GitLabRunnerToken: The token for your GitLab project. Get this from the "Runners" section of your GitLab project's "Settings --> CI / CD":
![Runner Token](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/runner-token.png)
* KeyName: Select the Key Pair that you earlier created. If the drop-down is empty, then you need to create a key pair in the current region.
* Subnet1ID, Subnet2ID, Subnet3ID, and VpcId: Select the VPC that you'd like to use, as well as the subnet for each Availability Zone where the GitLab Runner could exist. You'll want to use a private VPC for a real project, but for learning purposes you can use your account's Default VPC. If the region you're using doesn't have at least 3 availability zones, then you'll need to tweak the template to remove the usage of Subnet3ID--otherwise, just switch to a region that has 3 AZs.

Click "Next" twice (to use the default options). Then, click the checkbox to allow IAM resource creation, and click "Create".

Wait a few minutes for the stack creation to complete. You can now view the EC2 instance of the GitLab Runner in the console:

![GitLab Runner Instance](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/runner-ec2-instance.png)

You can also see, in GitLab, that there's a couple registered runners:

![Registered Runners](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/registered-runners.png)

Next, select the stack in the CloudFormation console, and click the "Outputs" tab:

TODO: ADD IMAGE of the OUTPUTs HERE

Copy the ARN of the RunnerIamRole to your clipboard. You'll need it for the next section when we set up the S3 backend, because we'll explicitly give that role permission to access the other account.

## Understanding the GitLab Runner Stack

At this point you have everything set up for running GitLab jobs that use Terraform.
Let's take a look at what's in the stack:

![CloudFormation S3 Backend](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/cf-gitlab-runner.png)

At the left of this diagram (above), we have an RunnerIamRole & RunnerInstanceProfile. These are
for giving the EC2 instance (that runs the GitLab Runner) permission to
assume the roles of the other accounts. The "AssumeRolePolicyDocument" allows the EC2 service to use the RunnerIamRole when it starts instances, and the "GitLabRunnerPolicy" allows the instance to assume the roles of the other accounts (i.e., "Staging" and "Production").
It's limited to assuming only "TerraformRole" and "S3BackendRole" (which are roles defined in "s3-backend.template") to avoid giving the instance too much access (e.g., it cannot assume the administrative roles of the DevOps account).
The CF template code for these is
as follows:

```
  RunnerIamRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: GitLabRunnerPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                  - 'sts:AssumeRole'
                Resource:
                  - 'arn:aws:iam::*:role/TerraformRole'
                  - 'arn:aws:iam::*:role/S3BackendRole'

  RunnerInstanceProfile:
    Type: 'AWS::IAM::InstanceProfile'
    Properties:
      InstanceProfileName: GitlabRunnerProfile
      Roles:
        - !Ref RunnerIamRole
```

After the IAM resources (RunnerIamRole & RunnerInstanceProfile), there's the
resources for starting the EC2 instance for the GitLab Runner: RunnerAutoScalingGroup
& RunnerLaunchConfiguration.  For these, the template code is a bit long, so
 you'll want to review it [directly in the file](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/gitlab-runner.template).
Of particular importance are the commands labeled as "1-Register-Docker-Runner" and "2-Register-Shell-Runner".
These commands each register a runner, associated with the specific GitLab project.
Why two? Technically you only need one of them, so you can decide which you'd like to keep
if you prefer not to use both. The difference is in which executor is used. The
"docker" executor is better at isolating jobs from one another (since it runs in
its own docker container), whereas the "shell" executor will have access to any
installed tools. If you look at the "UserData" section of the "RunnerLaunchConfiguration",
you'll see that we've installed docker, awscli, and terraform. This means that
your GitLab Pipeline Jobs can use any of these when they use the shell executor.
With the docker executor, you'd need an image that contains the tools. It's easy to create
such an image, and it's a good idea, but it's an extra step. For this tutorial,
we'll merely use the shell.  We'll have to make sure our job (in the .gitlab-ci.yaml file)
specifies a tag, either "terraform" or "awscli", in order for the runner to know
to use the shell runner--otherwise it'll use docker. This is because the shell
runner has the argument "--tag-list terraform,awscli" whereas the docker runner does not.

After the RunnerAutoScalingGroup & RunnerLaunchConfiguration, the "gitlab-runner.template"
continues with AutoScaling scheduled actions that will remove it at night and
re-create it in the morning. You can adjust the cron expressions if you prefer a
different schedule, or remove these altogether (if you want the instance to run 24x7).
Here's the code for these:
```
  ScaleDownAtNight:
    Type: 'AWS::AutoScaling::ScheduledAction'
    Properties:
      AutoScalingGroupName: !Ref RunnerAutoScalingGroup
      DesiredCapacity: 0
      MaxSize: 0
      MinSize: 0
      Recurrence: 0 2 * * *
    Metadata:
      'AWS::CloudFormation::Designer':
        id: d16f00fe-10db-4d20-bc66-0664546b8f33
  ScaleUpInMorning:
    Type: 'AWS::AutoScaling::ScheduledAction'
    Properties:
      AutoScalingGroupName: !Ref RunnerAutoScalingGroup
      DesiredCapacity: 1
      MaxSize: 1
      MinSize: 1
      Recurrence: 0 11 * * 1-5
```

If you use the scheduled actions, please pay special attention to the
next section, otherwise your GitLab project will accumulate a lot of
absolete runners (because they'll get freshly registered each morning but
never unregistered).

## Unregistration of the Runner

When you examine the "gitlab-runner.template" file, you'll notice a number of
resources that contain the line "Condition: HasGitLabApiToken". Each of these
is a component of the clean-up mechanism that will unregister the Runner
(from GitLab) whenever the EC2 instance terminates. You can enable unregistration
by:
1. Log-in to GitLab, and create an API Token (User Settings --> Access Tokens), as depicted below:
![CloudFormation S3 Backend](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/create-api-token.png)
1. Optional security feature:
    1. Use [KMS](https://aws.amazon.com/kms/) to create a customer-managed keypair
    1. Encrypt the GitLab API Access Token with it
    1. Tweak the Lambda function to decrypt it prior to use
    * Please keep in mind that the KMS keypair is not free. At the time of writing this article, it
    costs $1 per month. Encrypting this token is important because GitLab does not currently have
    any way to limit the scope of access, and anyone who has access to the Lambda function will
    be able to get the token. With use of KMS, you can separate who has access to view the Lambda
    function versus who has access to use the key for decryption.
    * The following screenshot (from the Lambda AWS Console) shows the helper for setting up KMS.
    After creating the stack, you can navigate to the Lambda function, enable the "helpers" and then
    click the "Code" button. It will give you JavaScript code for decrypting, which you can use for
    tweaking the Lambda function. You can then disable the helpers, and the IAM permissions so
    that only the Lambda function can decrypt with the key (and not the AWS Console user):
    ![KMS for Lambda Environment Variables](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/kms-helper-for-lambda.png)
    * You can find [the documentation here under "Environment Variable Encryption"](https://docs.aws.amazon.com/lambda/latest/dg/env_variables.html).
1. Update the CloudFormation stack, without changing the template, but just change the value of the GitLabApiToken parameter.

The presence of GitLabApiToken will cause the stack to look at follows (notice the entire lower section of this diagram is for support of unregistering the runner):
![CloudFormation S3 Backend](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/cf-gitlab-runner-full.png)

Here's how the unregistration works:
* The Autoscaling Group that controls the lifecycle of the EC2 instance will notify an SNS topic whenever an instance terminates.
* The SNS topic calls a Lambda function. The function makes an HTTPS request to GitLab (using the API token) to remove the runner registration.
* How does the Lambda function know which runner to unregister? It looks for the EC2 instance ID in the registration name. For this reason,
it's important that the runner-registration commands (see the RunnerLaunchConfiguration) include the EC2 instance ID in the name of the runner:
```gitlab-runner register --non-interactive --name Docker-Runner-$(ec2metadata --instance-id) ...```

You might be wondering why I used the GitLab API to remove the runner, rather than execute
a "gitlab-runner unregister" command--it's because I found that this command doesn't entirely
remove the registration on the GitLab server (they still remain in the list of runners, albeit inactive),
and it seems more reliable to use a lifecycle notification than a shutdown script
within the instance. However, I believe it could be done either way, each with pros/cons.

## Deploy the S3 Backend

In another account, go to the CloudFormation console and create a stack from the s3-backend.template file.

Click "Next", and then enter the missing parameter values:
* Stack name: the name of the stack, such as "Terraform-Backend"
* ExternalId: any random value, such as a new GUID. This is an extra security measure, that [AWS documentation explains](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html). It's arguably not needed for our usage since we own all accounts involved, however using it doesn't hurt. You'll need to use the same value when running Terraform (in the S3 Backend configuration).
* RunnerIamRoleArn: The ARN of the runner's role (previously copied into your clipboard when you looked at Outputs of the GitLab Runner stack).

Click "Next" twice (to use the default options). Then, click the checkbox to allow IAM resource creation, and click "Create".

Wait for the stack creation to complete. You now have everything set up for the S3 Backend. Let's take a look at what's in the stack:

![CloudFormation S3 Backend](https://raw.githubusercontent.com/pknell/gitlab-terraform/master/blog-images/cf-s3-backend.png)

## Understanding the S3 Backend Stack

TODO

## Create the Terraform and GitLab Configuration

Now that you have the GitLab Runner (with Terraform installed) and the S3 Backend, it's time to create some infrastructure! For this example, we'll just spin up an EC2 instance, but for your project it can be any AWS resources that Terraform supports.

TODO--present the main.tf

TODO--present the .gitlab-ci.yaml

## Run the Pipeline

