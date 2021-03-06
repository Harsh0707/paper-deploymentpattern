#### Thoughts on a continuous deployment strategy

Sometimes a wonderful tool comes along that makes kludgy processes radically better. [Packer](https://packer.io/)  and [ServerSpec](http://serverspec.org/) are great examples. The 1-2 punch of Packer + ServerSpec combined with the automation abilities of [Jenkins](https://jenkins-ci.org/) can make a significant impact on our automated server image creation. This combination has the possibilities of reducing our time-to-deploy, taking our visibility from *translucent* to *transparent*, improved our traceability, and generally made our Ops Engineers much, much happier. Without further a due, here is the proposal.

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

[Packer](https://www.packer.io/intro/) is a tool for creating server images for multiple platforms. It is easy to use and automates the process of creating server images. It supports multiple provisioners, all built into Packer.

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

```json
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
```

The configuration file can be divided into a few sections:

 * Variables
 * Builders
 * Provisioners

The variables section has all the variables that are used in the configuration file. These variables can be passed at runtime or can be predefined in a variable file. For example, variables can be passed at runtime in the Packer CLI command:

```bash
  packer build \
      -var "aws_access_key="    \
      -var "aws_secret_key="    \
      -var "user_name=`whoami`" \
      -var "aws_cert_path="     \
      -var "aws_key_path="      \
      -var "s3_bucket_name="    \
      -var "test_ami_flag="     \
      -var "source_ami_id="     \
      -var "product="           \
      -var "costcenter="        \
      -var "owner="             \
      -var "division="          \
  config.json
```

The variable file can be defined in a simple json format:

```json
  {
      "ami_name": "",
      "aws_account_id": "",
      "aws_region": "us-east-1",
      "ami_instance_type": "",
      "ami_description": "",
  }
```

The builders section defines the builder that will be used to create the server image. Since most of our servers are on AWS, we use the “Amazon EC2 (AMI)” builder. Others include:

 * __amazon-ebs__: Create EBS-backed AMIs by launching a source AMI and re-packaging it into a new AMI after provisioning
 * __amazon-instance__: Create instance-store AMIs by launching and provisioning a source instance, then rebundling it and uploading it to S3
 * __amazon-chroot__: Create EBS-backed AMIs from an existing EC2 instance by mounting the root device and using a Chroot environment to provision that device

We mostly use the “amazon-instance” builder, as most of our EC2 instances are instance-store backed. The provisioner section defines the provisioner that will be used to provision the server. Packer supports a variety of provisioners like:

* [Ansible](https://www.ansible.com/how-ansible-works)
* [Puppet Server](https://puppet.com/product/how-puppet-works)
* [Puppet Masterless](https://docs.puppet.com/references/glossary.html#masterless)
* [Chef Client](https://docs.chef.io/chef_client.html)
* [Chef Solo](https://docs.chef.io/chef_solo.html)
* [SaltStack](https://saltstack.com/saltstack-enterprise-cloudops/)
* [Custom Provisioner](https://www.packer.io/docs/extend/provisioner.html)

<sub>List of all [Provisioners here]()</sub>


Since we heavily use Puppet for provisioning our servers, Packer’s Puppet Server provisioner is the logical choice for us. While using the Puppet Server provisioner I believe Puppet pattern for **roles** and **profiles** can be easily integrated with Packer. Most of our Puppet Modules are shared across various projects, which makes it interesting to work with - these shared modules will be included in the Puppet Server provisioner configuration on registration with the master.


#### Provisioning Logs

During the provisioning process, the logs that we are generating with Puppet can be stored on the server image for debugging purposes. This is done by enabling Puppets logging capabilities by customizing `/etc/default/puppet`. By adding the following:

```bash
  # Startup options
  DAEMON_OPTS="--logdest /var/log/puppet/puppet.log"
```

`/etc/default/puppet` is then sourced by `/etc/init.d/puppet`, so the options you added here will be executed when puppet service is started.

We can even go as far as adding in custom log rotation:

```bash
  /var/log/puppet/*log {
    missingok
    sharedscripts
    create 0644 puppet puppet
    compress
    rotate 4

    postrotate
      pkill -USR2 -u puppet -f 'puppet master' || true
      [ -e /etc/init.d/puppet ] && /etc/init.d/puppet reload > /dev/null 2>&1 || true
    endscript
  }
```


#### Benefits of using Packer

There are various benefits of using Packer in terms of performance, automation, and security:

 * Packer spins up an EC2 instance, creates temporary security groups for the instance, creates temporary keys, provisions the instance, creates an AMI, and terminates the instances - and it’s all completely automated
 * Packer uploads all the Ansible playbooks and associated variables to the remote server, and then runs the provisioner locally on that machine. It has a default staging directory (/tmp/packer-provisioner-ansible-local/) that it creates on the remote server, and this is the location where it stores all the playbooks, variables, and roles. Running the playbooks locally on the instance is much faster than running them remotely.
 * Packer implements parallelization of all the processes that it implements
 * With Packer we supply Amazon’s API keys locally. The temporary keys that are created when the instance is spun up are removed after the instance is provisioned, for increased security.
 * Testing Before Creating Server Images

Packer helped solve the problem of automating server image creation. We still thought there was room for improvements in terms of testing whether the instance was provisioned correctly, so we began using ServerSpec.


#### What is ServerSpec?

ServerSpec offers RSpec tests for your provisioned server. RSpec is commonly used as a testing tool in Ruby programming, made to easily test your servers by executing few commands locally. We can then write some simple tests using ServerSpec that would help indicate whether the instance was ready to be imaged:

 * Testing services that make up our web server such as Nginx, PHP-FPM, various routers, etc
 * Testing common monitoring and alerting services such as Sensu and DynaTrace
 * Testing ports that various services run on


#### Integrating ServerSpec with Packer

In order for us to decide whether the server image is created or not, ServerSpec can be integrated with Packer right after the provisioning is complete. This is done by using Packers `shell` and `file` provisioners. First, we will need to create a temporary directory on the server and copy the ServerSpec tests to be run based on server type, then run a simple bash script that kick off the ServerSpec tests locally on that machine.

The Packer configuration file had the following defined in order to run ServerSpec tests:

```json
  {
      "type": "shell",
      "inline": [
          "mkdir /tmp/tests"
      ]
  }, {
      "type": "file",
      "source": "serverspec_tests/server-type/",
      "destination": "/tmp/tests"
  }, {
      "type": "shell",
      "script": "scripts/serverspec.sh"
  }
```

If the tests pass, then Packer will go ahead and image the server then create an AMI. However, if a tests fail, Packer will receive an error code from the shell provisioner that would terminate the image creation process and the instance. This is the bash script we used for installing and running the ServerSpec tests:

```bash
  #!/usr/bin/env bash -ex
  # serverspec.sh - RSpec tests for servers
  echo "Installing Serverspec"
  sudo gem install serverspec

  # move and set rake-handler
  # bypasses /opt/sensu/embedded/bin/rake vs /usr/bin/rake etc
  cd /tmp/tests ; rake=$(which rake |head -n1 |awk '{print$3}')
  echo "Running integration tests for AMI"
  $rake spec
```

This process allowed us to be confident about the images that were being built using Packer.


#### Automation Using Jenkins

Jenkins is used to automate the process of creating and testing images. A Jenkins job can be parameterized to take inputs such as project name, username, Amazon’s API keys, test flags,  etc., which will allow our engineers to build project specific image rapidly without installing Packer and its CLI tools. Jenkins will take care of the AMI tagging, CLI parameters for Packer and notifications to the our team about the status of the Job:

![Pipeline](https://raw.githubusercontent.com/ehime/Deploy-Strategy/master/assets/automation-pipeline.jpg "Pipeline")


#### In the Pipeline
There’s still room for improvement with regards to image creation. Still in the pipeline are:

 * Automatically triggering the Jenkins job on git commit
 * Creating an AMI management tool for creating and deleting server images
 * Running continuous sanity checks on our EC2 servers to identify any drift in configuration
