## Kubernetes, Docker Swarm and AWS ECS: Pros and Cons

In my [previous article](https://otseobande.com/setup-docker-swarm-on-digital-ocean-with-terraform-ck6zh46vv05qndfs1bl4n2w3j), I started by stating that each container orchestration platform has pros and cons. In this article, I'll be exploring the pros and cons of Kubernetes, Docker Swarm and AWS ECS (Elastic Container Service).

## Kubernetes

> Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

Kubernetes prides itself as a production workload ready open-source platform serving Google teams for 15 years. This explains all the hype around this particular tool. Let's look at it's pros and cons:

### Pros
- **Vendor Agnostic:** One of the well marketed characteristics of Kubernetes is it's open-source nature. This makes it usable on any cloud provider and different cloud providers are beginning to offer managed Kuberentes services. So it's very hard to ignore Kuberentes as a tool of choice.
- **Monitoring:** Kubernetes has self monitoring which can be used to get the status of resources in a kubernetes cluster and dig into very specific resource information. It's also very easy to use tools like the CLI, the Kubernetes Dashboard or third party tools like [Prometheus](https://prometheus.io/) along with [Grafana](https://grafana.com/) to get full monitoring of your workload cluster. 
- **API:** Kubernetes has a very robust API that can be easily integrated with other software to create powerful automation. It also has client libraries in Go, Python, Java and Javascript as of the time of writing this article.
- **Horizontal Scaling:** Kubernetes supports running a cluster on multiple nodes (machines). So the growth capabilites for a Kubernetes cluster is huge.
- **Automatic rollouts and rollbacks:** This is a life saving feature of Kubernetes. Once a deployment fails or times out, Kubernetes automatically rolls back the deployment with no downtime to your application consumers.
- **Automatic container balancing across nodes:** Kubernetes automatically manages how nodes are distributed across the cluster nodes based on available resources and CPU and Memory requests configured for the cluster.
- **Configuration:** Kubernetes allows declarative configuration of resources using YAML and this enables managing your resource configuration as code.
- **Deployment strategies:** Kubernetes supports a number of deployment strategies that ensure application availability across multiple deployments. It uses rolling-update by default.

### Cons
- **Learning curve:** As versatile as Kubernetes is, it can be very challenging learning all of it's concepts and how it is setup.
- **Ease of setup:** Managed Kubernetes services like Elastic Kubernetes Service (EKS), DigitalOcean Kubernetes, Google Kubernetes Engine (GKE), along with other tools have made this a lot easier but setting up Kubernetes on bare metal can still be challenging.
- **Cost:** For some applications, Kubernetes might be a more expensive solution to use.


## Docker Swarm

Docker Swarm is Docker's solution to container orchestration. It enables running replicated docker containers as services across cluster Docker Engines that make up a **Swarm**. 

### Pros 
- **Learning Curve:** Once you understand how Docker works it's easy to spin up a docker swarm as it uses the same Docker CLI tool.
- **Ease of setup and Cost:** Compared to Kubernetes, Docker Swarm is easier to setup and is relatively cheaper.
- **Configuration:** Docker Swarm stack files use a similar docker-compose YAML configuration with extra properties and should be straight forward for anyone with docker-compose knowledge.
- **Horizontal Scaling:** Docker Swarm supports creating a swarm with multiple machines. This supports horizontal scaling for applications.
- **Simplicity:** Docker Swarm has a small feature set which is quite easy to grasp.

### Cons
- **Monitoring:** Monitoring on Docker Swarm is not as straight forward as other solutions. It requires some work to setup.
- **Deployment strategies:** Docker Swarm gives little to no control over deployments using deployment strategies. Although, some smart techniques can still be used.
- **Random container placement:** I found that sometimes the automatic container placement across Docker Engines is as optimal that of other solutions. Although, this can be easily managed using resource constraints.
- **Simplicity:** Sometimes some more flexibility is need and Docker Swarm might feel limiting in such a case.

## AWS Elastic Container Service (ECS)

> Amazon Elastic Container Service (Amazon ECS) is a fully managed container orchestration service.

AWS is a beast and when they provide a service they do it well. I'm not saying their services are perfect but they are very solid and AWS ECS is no exception. 

### Pros
- **Ease of setup:** An ECS cluster can be setup in about 5 minutes because it's a managed service. It's good to note that managed Kubernetes services also benefit from this kind of ease of setup.
- **Learning Curve:** I think ECS is quite easy grasp (this is relative)
- **Integration with other AWS services:** ECS has butter smooth integration with other great AWS services like CloudWatch, Elastic Load Balancers (ELB), Auto Scaling Groups (ASG), CodeDeploy etc. This makes it a very robust service.
- **Serverless Offering:** ECS offers serverless container orchestration as a service called [Fargate](https://aws.amazon.com/fargate/). How cool is that!?

### Cons
- **It's only available on AWS:** ECS is vendor specific and can only be used on AWS.
- **Closed-source:** Some users consider the service being closed sourced as a disadvantage.

So there you have it, these are the pros and cons of these container orchestrations platforms I have observed from my experience and is not intended to be an exhaustive list of what these platforms can or cannot do. You can use any tool of your choice as the decision of the best tool to use always depends on the context it's being used in.

#### Resources
- https://www.criticalcase.com/blog/kubernetes-features-and-benefits.html
- https://us-east-2.console.aws.amazon.com/ecs/home?ad=c&cp=bn&p=ecs&region=us-east-2#/firstRun
- https://docs.docker.com/engine/swarm/
- https://kubernetes.io/