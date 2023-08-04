---
layout: post
title:  "Could we have two EKS clusters sharing the same VPC ?"
date:   2023-08-03  11:55:00 +1000
categories: aws
image: /assets/images/multi-cluster-eks-vpc.png
slug: eks clusters sharing same vpc
tags: aws eks cluster-upgrades mutiple-cidr-blocks
---

Kubernetes upgrades always have been a risk item in anyones change log , you would have alredy heard enough stop the world stories . One of our customers when upgrading EKS cluster to newer versions wanted to completely avoid a down time scenario due changes which might break the cluster . The customer was a fintech trading organisation where users of their app span globally and continously use their apps and api`s throughout the day ,which means they could not risk a downtime whatsoever . Transactions amounting to millions are being executed via micros-services each day, a little down cost them dearly.

When it comes to Kube some upgrades are straight forward but some are not due to various reasons. For instance , if you have various other components you rely on like service meshes might not work or fail due to version mis-match issues with the kubernetes. Also some k8 upgrades might introduce deprecations of the previous apis and the add ons and the plugins rely on those api versions might fail to work after the upgrade . Upgrades are envitable , we can't simply ignore it , thats how we keep the cluster upto date with the latest standards and apis versions and security fixes . 

EKS vpc had three CIDR ranges attached to it where the managed/self managed nodes primary interfaces will reside and the rest of the CIDR ranges were dedicated to the pods . This would allow them to have thousands of pods to be operated having its own IP from the a dedicated VPC range. Public ALB was fronting all the microservices in operation and it was managed seperately not via the kubernetes ingress controller.

The main database serving the stateless microservices resides in the same VPC and also few other services such as AWS managed kafka service and few stateless services some are app specific ,rest of them were Kubernetes Add ons which require stateless deployments ( prometheus , elastisearch so on ). 

s
 