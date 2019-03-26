---
title: "Terraform Digitalocean Kubernetes"
date: 2019-03-26T08:30:04-04:00
tags: ["devops"]
draft: true
---


DigitalOcean recently launched managed kubernetes on their cloud which was really interesting to me.   Doing some 
rough math it looked around 30-50% cheaper than the big three (AWS, GCP, Azure).  DigitalOcean is considerably
simpler than AWS which is great for small teams like ours.   In addition I really liked the
idea of kubernetes instead of vendor lock in like fargate or elasticbeanstalk because I could more easily switch
to another cloud provider or even use two of them in the future.

One of the goals was to automate the infrastructure through code.   So I set out to use terraform to do that and I'm
really happy with the kubernetes integration.

You can check out the project from [github here](https://github.com/dcardamo/terraform-k8s-do)

To use it you need to:
1.  Install [terraform](https://terraform.io)
2.  git clone https://github.com/dcardamo/terraform-k8s-do
3.  cd terraform-k8s-do/prod
4.  edit prod.tfvars and give it the settings you want including your digitalocean API token
5.  terraform init
6.  bring up your cluster:  terraform apply -var-file prod.tfvars
7.  Each time you want to update your cluster:  terraform apply -var-file prod.tfvars

Make sure to backup your tfstate files after each apply.  You'll need those for the next time you
run terraform.

