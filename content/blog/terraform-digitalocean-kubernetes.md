---
title: "Terraform Digitalocean Kubernetes"
date: 2019-03-26T08:30:04-04:00
tags: ["devops"]
draft: false
---


DigitalOcean recently launched managed kubernetes on their cloud which was really interesting to me.   Doing some
rough math it looked around 30-50% cheaper than the big three (AWS, GCP, Azure).  DigitalOcean is considerably
simpler than AWS which is great for small teams like ours.   In addition I really liked the
idea of kubernetes instead of vendor lock in like fargate or elasticbeanstalk because I could more easily switch
to another cloud provider or even use two of them in the future.

One of the goals was to automate the infrastructure through code.   So I set out to use terraform to do that and I'm
really happy with the kubernetes integration.

You can check out the project from [github here](https://github.com/dcardamo/terraform-k8s-do)

Using it you'll get the following features:

* Pick your cluster size, node size, region
* Choose your version of kubernetes provided DigitalOcean supports it
* Helm installed
* Traefik installed as the ingress service
  * DigitalOcean load balancer in front of the redundant Traefik services
  * Lets encrypt SSL certs using your domain.   Requires you host your DNS with digitalocean
* Kubernetes Dashboard installed
* Gitlab account created for continuous deployment
* Easy to add more clusters such as staging/testing/etc

To use it you need to:

1.  Install [terraform](https://terraform.io)
2.  git clone https://github.com/dcardamo/terraform-k8s-do
3.  cd terraform-k8s-do/prod
1.  create a file called `prod.tfvars` with inputs matching `vars.tf`.  Example below
2.  `terraform init`
3.  `terraform apply -var-file=autobots.tfvars`
4.  `cp kubeconfig.yaml ~/.kube/config`

Here is an example prod.tfvars:

{{<highlight terraform "linenos=table">}}
cluster_name = "production"
k8s_version = "1.13.4-do.0"
node_size = "s-1vcpu-2gb"
cluster_size = 2
do_zone = "nyc1"
traefik_replicas = 2
do_token = "your_do_token"
lets_encrypt_email = "me@example.com"
lets_encrypt_main_domain = "example.com"
{{</highlight>}}

Make sure to backup your tfstate files after each apply.  You'll need those for the next time you
run terraform.

### DigitalOcean Kubernetes Cluster
![kubernetes cluster](/img/terraform-digitalocean-kubernetes/terraform-digitalocean-kubernetes1.png)

### DigitalOcean Load Balancer
![kubernetes cluster](/img/terraform-digitalocean-kubernetes/terraform-digitalocean-kubernetes2.png)

Congratulations, you have a kubernetes cluster running with infrastructure as code.  The setup I'm showing here costs $20 for the worker nodes and $15/mo for the load balancer.   Pretty slick price in my opinion.
