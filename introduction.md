[//]: (I just launched my new platform on AWS, trying to leverage most of the best practices when it comes to cloud architecture and scalability. This platform is mostly used for proof of concepts and prototypes - and since today also as a public blog to share the experiences I made with this environment and the prototypes I've built.)
[//]: ("docker": "1.10")
[//]: ("kubernetes": "1.4.3")
[//]: ("CoreOS": "1122.2.0")
[//]: ("Puppet": "4.6.0")
I just launched my new platform on AWS, trying to leverage most of the best practices when it comes to cloud architecture and scalability. This platform is mostly used for proof of concepts and prototypes - and since today also as a public blog to share the experiences I made with this environment and the prototypes I've built. 

## The components that power the automation pipeline
You need different layers of automation if you want to build a "fully automated environment". There's more than just spinning up machines. It's all about your cloud design, service provision and applications from build to deployment. With nowadays possiblities, you can build very complex cloud setups with just one command. The most important part is to use the right technology for your needs - and wire them correctly. 

### Cloud automation
At the bginning, you need something to manage your Cloud Provider automatically - to spin up a VPC, setup the networking, routing and logically place some autoscaling groups there. This is where your Cloud Architecture lives.

I'm using [terraform](https://www.terraform.io) for this. Terraform doesn't support all cloud providers equally, but the support for AWS, GCP and Azure is quite useful - whereas support for vCloud & vSphere is there... but not usable at all.

To achieve machine failure resilience, I never spin up instances. All machines are kicked off via autoscaling groups. Even if min and max sizing is 1, but this guarantees that amazon will notice that 1 machine crashed and will fire up a new one. You just pay for the particular hour 2 instances instead of 1. 

### Service provisioning
After the machine came up, some provisioners need to run in order to setup services like haproxy or jenkins. To hook into the boot process of the machines I use [Cloud-Init](https://cloudinit.readthedocs.io/en/latest/) for my ubuntu instances, provided by the [user_data](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html#user_data) argument to the [aws_launch_configuration](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html) resource.

Cloud-Init does a bit of the basic setup (like setting timezones, charsets, etc) and also exports some [facts](https://docs.puppet.com/puppet/latest/reference/lang_facts_and_builtin_vars.html) for [puppet](https://docs.puppet.com/puppet/4.0/reference/index.html). These facts are exported from the naming scheme I defined. It simply looks like:

```
$environment-$service
production-haproxy
```
The environment is not staging or production (well, it could be...), it identifies the source where it can find the puppet profiles/roles/modules - in a defined S3 bucket. So having multiple environments (let's say production and staging) looks like this in an S3 bucket:
```
my-bucket-name/
  puppet-staging.tar.gz
  puppet-production.tar.gz
```
The environments reflect different development branches on the puppet git repository, to be able to branch features without crashing the stable parts of the environment.

Cloud-Init exports this information and starts a puppet run - with all the sources fetched from S3.
For the above example, it will fetch `puppet-production.tar.gz`, extracts it and applies the role `haproxy` to this instance.

You might wonder, why I don't use tags for this? Well, I did - and I will continue to do so, if terraforms [aws_spot_fleet_request](https://www.terraform.io/docs/providers/aws/r/spot_fleet_request.html) supports tags in the future (Please note: this resource is only available in terraform 0.7 and above).

Just to be clear... I only have 2 instances running on ubuntu. Most of the instances are part of my [kubernetes](http://kubernetes.io/) cluster, powered by [CoreOS](https://coreos.com/). If you have a fleet of machines, only existent to run (docker) containers - what is better than running them on an operating system, that is designed to run containers and provide a very low system level overhead? Right...

But since there's not a real benefit of having puppet-managed CoreOS machines, I used CoreOS' own [cloud-config](https://coreos.com/os/docs/latest/cloud-config.html) implementation for service (or unit) provisioning. Again provided by [user_data](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html#user_data) from terraform. Those machines are more or less stateless at all, so I don't care about "operational provisioning" (cloud-config only runs once, not frequently like puppet). Those machines are even doing a silent OS update and reboot while being in production (locked via etcd that only 1 machine updates in time). The only special thing I did for maintaining them is to split up the fleet into 2 autoscaling groups. This way I can destroy and change 1 of the 2 without affecting my uptime.

I really learned to love CoreOS over the last months - you can expect a couple of articles about CoreOS in the future!

## Everything about containers
As you might asume if you've read the post until here, I'm heavily relying on containers. Containers are mainly used as a deployment artifact, but also to **couple** the service setup with the applications. Think about something like PHP. You configure your webserver with a domain name, a document root and a link to FPM. You do that again for a second service. Than you have 1 nginx, sharing the responsibility for 2 (or even more) PHP applications. This is how you build monoliths.

### The container infrastructure
As probably all enterprise grade container platforms, I needed a private container registry. There are many cool hosted solutions around - but since I wanted to implement everything myself, I decided to go with [Suse Portus](http://port.us.org/). 

I'm running a 3 master kubernetes setup, loadbalanced via haproxy, and etcd running on the master nodes - clustered using the [public discovery service](https://coreos.com/os/docs/latest/cluster-discovery.html). The apiserver can be used from all nodes, whereas only 1 `scheduler` and 1 `controller` is allowed to run at the same time - but since they introduced `--leader-elect=true` for those 2 components, the clustering became very easy.

For building the containers from source, I use [jenkins](https://jenkins.io/). Why? Good question... I think it's more the fact that I'm used to it than that I like it ... 


### The build pipeline
My default setup is fairly easy:
```
[Github] --> [Jenkins] --> [Portus] <--> {Kubernetes}
```
The Jobs and endpoints differ of course, but even the logic per Jenkins Job is almost always the same:
```
make docker
make push
```
I ship everything with a `Makefile`, it doesn't matter if it's a react js application or a go-service. Everything that needs to be handled via Jenkins have these targets. Application specific build can still be handled via these targets, I simply change them - since they're copies, it doesn't hurt and I have a well known starting point.

### Container Deployment
The registry is obviously connected to my kubernetes cluster, to power the application deployment. Since I'm running K8s > 1.3, the first choice was to go with [Deployments](http://kubernetes.io/docs/user-guide/deployments/) "for general purposes".
My manifests for kubernetes live in a git repository and are automatically deployed with a systemd-unit at cluster startup time. 

## Monitoring and logging
For such a flexible and self-managing environment, monitoring and logging is one of the most important topics. But e.g. implementing a full blown [ELK] with a good sizing and failover setup is quiete expensive in AWS. That's why I'm using [Logz.io] - it's relativly cheap and comes with a powerful setup per default. It has also some great AWS goodies - and they're not only exposing a dashboard, but pretty much all the components of the ELK stack.

For infrastructure monitoring, I decided to give google's [stackdriver] a chance. It was great for the beginning, easy to setup - but only limited to data, that can be queried via the CloudWatch API. I haven't tried the custom metrics so far, but not even being able to see the memory usage right away is a bit less than what I expected (memory usage isn't a default metric from CloudWatch).

For Container monitoring, I use [heapster]. Straight forward, easy to integrate into k8s, just works. 

## The Blog architecture
I'm gonna write about my blog architecture? Really? 
Don't blame me about it, I'm also tired of the "regular ideas" of running a blog/cms (and I wanted to have a lightweight but powerful reference implementation) - so you can expect something that's worth writing about ;-)

### Involved components
Because the CRUD stuff really bores me in web contexts, I've build a more relyable architecture and simply "micro-serviced" the blog part. Since every service can have it's own architectural design implementation (due to the power of containers/kubernetes/go-micro), this part just explains the technical components involved for this single, small unit.

[Github repository]
> This repository is used to store my blog articles

[Github SNS hook]
> This hook is used to publish an event to the defined sns_topic

[AWS Lambda]
> A lambda function gets triggered via previous SNS topic

[AWS S3]
> The lambda function converts the Markdown to JSON, uploads these JSONs and pictures to an S3 bucket & publishes an event

[NATS]
> The event from AWS Lambda ends up as a message on NATS - this message forces my microservice to reload its blog content in memory, because the data in S3 has changed

[Go-Micro]
> The go-ecosystem I'm using for writing my services

[ReactJS]
> Frontend implementation

### Architecture overview
![Blog Architecture][Blog Architecture]

Everything starts with a small notification from github to AWS SNS. AWS Lambda listens on this topic and starts a go tool which clones the github repository. This go tool converts the Markdown files to JSON, including some extracted meta data from the Markdown comments. It writes those JSONs to the filesystem and uploads them, and the pictures, to an S3 bucket. After the upload is finished, it publishes an event to NATS, which forces my blog service to update its data in memory. ReactJS then consumes the blog-service and shows you this article. 

This blog acted more as a proof of concept for a fully integrated platform, leveraging most of its capabilities. As a side effect, I'd accept a nomination of the most overengineered blog ever :D

[Go-Micro]: https://micro.mu/
[ReactJS]: https://facebook.github.io/react/
[NATS]: http://nats.io/
[AWS S3]: https://aws.amazon.com/s3/
[AWS Lambda]: https://aws.amazon.com/lambda/details/
[Github SNS hook]: https://github.com/github/github-services/blob/master/docs/amazonsns
[Github repository]: https://github.com/iwalz/blog
[Blog Architecture]: img/introduction/blog_architecture.png
[ELK]: https://www.elastic.co/de/webinars/introduction-elk-stack
[Logz.io]: http://logz.io
[stackdriver]: http://www.stackdriver.com/
[heapster]: https://github.com/kubernetes/heapster