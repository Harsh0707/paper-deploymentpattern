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
