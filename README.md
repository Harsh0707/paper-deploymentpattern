#### Thoughts on a continuous deployment strategy

Sometimes a wonderful tool comes along that makes kludgy processes radically better. [Packer](https://packer.io/)  and [ServerSpec](http://serverspec.org/) are great examples. The 1-2 punch of Packer + ServerSpec combined with the automation abilities of [Jenkins](https://jenkins-ci.org/) can make a significant impact on our automated server image creation. This combination has the possibilities of reducing our time-to-deploy, taking our visibility from ‘translucent’ to ‘transparent’, improved our traceability, and generally made our Ops Engineers much, much happier. Without further a due, here is the proposal.

![Packer + ServerSpec + Jenkins](https://raw.githubusercontent.com/ehime/Deploy-Strategy/master/assets/packer-serverspec-jenkins.jpg "Packer + ServerSpec + Jenkins")
