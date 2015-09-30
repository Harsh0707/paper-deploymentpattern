#### Thoughts on a continuous deployment strategy

Sometimes a wonderful tool comes along that makes kludgy processes radically better. [Packer](https://packer.io/)  and [ServerSpec](http://serverspec.org/) are great examples. The 1-2 punch of Packer + ServerSpec combined with the automation abilities of [Jenkins](https://jenkins-ci.org/) can make a significant impact on our automated server image creation. This combination has the possibilities of reducing our time-to-deploy, taking our visibility from ‘translucent’ to ‘transparent’, improved our traceability, and generally made our Ops Engineers much, much happier. Without further a due, here is the proposal.

![Packer + ServerSpec + Jenkins](https://raw.githubusercontent.com/ehime/Deploy-Strategy/master/assets/packer-serverspec-jenkins.jpg "Packer + ServerSpec + Jenkins")

One of the advantages of having our application run in the cloud is that we can rapidly provision and decommission servers that the application runs on. At [McGraw-Hill Education](https://www.mheducation.com), we love AWS, but could better our practice by creating and replacing EC2 instances all the time. This is a fairly easy process using Server Images ([AMIs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)) but can be a tedious task if you provision them manually. Hence, there is a need to create Server Images rapidly and in a way that the images are tested in order to guarantee a healthy cloud infrastructure.


##### The Problem

Most of our servers are in EC2 and we use Puppet to provision those servers. To create a server on EC2 we have two processes, the first of which involved using Puppet to provision the server:

 * Spin up an EC2 instance using our hardened base images
 * Run an Puppet Enterprise to provision the instance
 * Do a manual sanity check for all the services running on the instance

This process is slow, and requiring manual steps is not ideal.

The second process involved using a server image (AMI) and spinning up an EC2 instance. This process is faster, but isn’t without problems: building the server image is slow, and there are no checks in place to test the image before use. Additionally, there were no provisioning logs for debugging the server image, which made it difficult for our Operations Engineers to troubleshoot. Creating server images looked like this:

![Process](https://raw.githubusercontent.com/ehime/Deploy-Strategy/master/assets/process.jpg "Process")

I am personally not happy with potential for errors here, so lets rethink this entire process using Packer.

##### What is Packer?

Packer is a tool for creating server images for multiple platforms. It is easy to use and automates the process of creating server images. It supports multiple provisioners, all built into Packer.

##### Why Packer?

In order to simplify the steps involved in creating a server image, we choose Hashicorp’s Packer. These were a few simple reasons why Packer was the obvious choice:

 * Supports multiple platforms such as Amazon EC2, OpenStack, VMware and VirtualBox
 * It’s easy to use and is mostly automated
 * It supports Puppet Enterprise as a provisioner, and we at MHE LOVE Puppet!
 * It’s written in Go and is created by the same guys that did Serf, Consul, and Vagrant (which are amazing)

#### Challenges with Packer
We encountered two specific challengers with Packer we will need to overcome:

 * Testing the Packer-built server images
 * Automating the build and test process

#### The Implementation strategy

The implementation of Packer will be a very simple one (compared to things I've been doing with Packer in the past that is). Packer needs a configuration file that has all the information needed to create a server image. We also used a variables file that has a combination of variables that the configuration file needs, along with custom variables for tagging the server image. Here’s an example Packer configuration file:

    {
      "variables": {
        "ami_name": "",
        "aws_access_key": "",
        "aws_secret_key": "",
        "user_name": "",
        "test_ami_flag": "",
        "aws_cert_path": "",
        "aws_key_path": "",
        "source_ami_id": "",
        "aws_account_id": "",
        "aws_region": "",
        "ami_instance_type": "",
        "ami_description": "",
        "s3_bucket_name": "",
        "product": "",
        "costcenter": "",
        "owner": "",
        "division": ""
      },
      "builders": [{
        "type": "amazon-instance",
        "access_key": "{{user `aws_access_key`}}",
        "secret_key": "{{user `aws_secret_key`}}",
        "account_id": "{{user `aws_account_id`}}",
        "region": "{{user `aws_region`}}",
        "source_ami": "{{user `source_ami_id`}}",
        "instance_type": "{{user `ami_instance_type`}}",
        "ssh_username": "ubuntu",
        "ami_name": "{{isotime |clean_ami_name}} {{user `ami_name`}}",
        "s3_bucket": "{{user `s3_bucket_name`}}",
        "x509_cert_path": "{{user `aws_cert_path`}}",
        "x509_key_path": "{{user `aws_key_path`}}",
        "security_group_ids": [
          "all",
          "default"
        ],
        "tags": {
          "Product": "{{user `product`}}",
          "CostCenter": "{{user `costcenter`}}",
          "Owner": "{{user `owner`}}",
          "Division": "{{user `division`}}",
          "author": "{{user `user_name`}}",
          "test_ami": "{{user `test_ami_flag`}}",
          "packer": "true",
          },
        "ami_description": "{{user `ami_description`}}",
        "bundle_upload_command": "sudo -n ec2-upload-bundle -b {{.BucketName}} -m {{.ManifestPath}} -a {{.AccessKey}} -s {{.SecretKey}} -d {{.BundleDirectory}} --batch --retry"
      }],
      "provisioners": [{
         "type": "puppet-server",
         "options": "--test --pluginsync",
         "facter": {
           "server_role": "webserver"
         }
      }]
    }
