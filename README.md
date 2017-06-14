# aws-ami-builder

aws-ami-builder is a sample project that leverages [AWS CodeBuild](https://aws.amazon.com/codebuild/), [HashiCorp Packer](https://www.packer.io/intro/index.html), and [Ansible](https://www.ansible.com/) to automate the process of baking Amazon Machine Images (AMI).

In this example, we build a simple web server AMI that builds on the latest version of the [Amazon Linux AMI](https://aws.amazon.com/amazon-linux-ami/), adding [nginx](https://nginx.org/en/) and the AWS CloudWatch agent. In a larger project, you could utlize this baseline in a number ways, such as including your own standard security packages or building other AMI flavors.  See [AWS AMI Design](https://aws.amazon.com/answers/configuration-management/aws-ami-design/) for other considerations when baking AMIs.

## Getting Started

To get started, clone this repository locally:

```
$ git clone https://github.com/jkahn117/aws-ami-builder.git
```

The repository contains a [CloudFormation](https://aws.amazon.com/cloudformation/) that will launch the needed AWS resources as well as source code for the sample.

### Prerequisites

To run the aws-ami-builder sample, you will need to:

1. Select an AWS Region into which you will deploy services. Be sure that all required services (AWS CodeCommit and AWS CodeBuild) are available in the Region you select.
2. Confirm your [installation of the latest AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html).
3. Confirm the [AWS CLI is properly configured](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-quick-configuration) with credentials that have administrator access to your AWS account.
4. Git client installed and configured as well as a basic familiarity with Git.

## Create AWS Resources with CloudFormation

We will use AWS CloudFormation to create a CodeCommit Repository, CodeBuild Project, and a CodeBuild service role. Additionally, we will create a sample Role that can be attached to an EC2 instance if you choose to launch an instance using the baked AMI.

Again, please be sure the AWS CLI (see `aws configure`) is configured to use an AWS Region in which CodeCommit and CodeBuild are available. You can also utilize the AWS Console to complete the following.

```
$ aws cloudformation create-stack \
        --stack-name    aws-ami-builder \
        --capabilities  CAPABILITY_NAMED_IAM \
        --template-body file://aws-ami-builder.cloudformation.yml
```

The CloudFormation stack will require several minutes to finish creation.  Once complete, we can retrieve the URL for the CodeCommit repository as follows:

```
$ aws cloudformation describe-stacks \
        --stack-name    aws-ami-builder \
        --query         "Stacks[].Outputs[?OutputKey=='CodeCommitRepositoryHttp']"
```

## Commit Code and Build AMI

Next, configure your [Git client to interoperate with the newly created CodeCommit repository](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up.html). Once complete, we will add the CodeCommit repository as a remote for this directory and push code to it.

If you are planning to use your IAM credentials to access CodeCommit, you may want to use the AWS CodeCommit Credential Helper ([Windows](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-windows.html), [Linux/macOS](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html)).

```
$ git remote add codecommit <CODECOMMIT_REPO_HTTP_URL>

$ git push codecommit master
```

With the sample code now in CodeCommit, we will start a build using CodeBuild:

```
$ aws codebuild start-build \
        --project-name  aws-ami-builder
```

The build will require several minutes. The AWS Console provides a nice overview of the build progress as well as detailed logging.

As part of the build process, Packer will create a new AMI in your account.  We can query for a listing of custom AMIs in your account as follows (the AMI will have a name like `webserver_amazon_linux_<TODAYS_DATE>`):

```
$ aws ec2 describe-images \
        --query         "Images[?contains(Name, 'webserver_amazon_linux')].ImageId"
```

## Launching a Web Server Instance

Finally, we can test our new web server image by launching a new EC2 Instance using the AMI ID captured in the previous step.  We will leave [launching the instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/launching-instance.html) and navigating to it in a browser to you, but you should be greeted with an nginx on Amazon Linux test page.

## Cleaning Up

When ready, it is easy to remove all resources created in this sample via CloudFormation:

```
$ aws cloudformation delete-stack \
        --stack-name    aws-ami-builder
```

## Further Reading

If you are interested in diving deeper on approaches to building AMIs, the following articles may be of interest:

* [AWS AMI Design](https://aws.amazon.com/answers/configuration-management/aws-ami-design/)
* [AMI Builder with CodeBuild and Packer](https://aws.amazon.com/blogs/devops/how-to-create-an-ami-builder-with-aws-codebuild-and-hashicorp-packer/)

As a next step, you may also consider using tools such as [Amazon EC2 Systems Manager to automate deployment of your newly created AMI](https://aws.amazon.com/blogs/aws/streamline-ami-maintenance-and-patching-using-amazon-ec2-systems-manager-automation/).

### Our Packer Template

One aspect of our Packer template (`webserver.packer.json`) that may require some explanation is the [`source_ami_filter`](https://www.packer.io/docs/builders/amazon-ebs.html#source_ami_filter) section.  In this case, we are filtering the list of available AMIs for the latest version of the Amazon Linux AMI owned by Amazon, backed by EBS GP2 storage.

## Authors

* **Josh Kahn** - *Initial work* (jkahn@)
