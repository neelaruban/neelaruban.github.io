---
layout: post
title:  "Could we have two EKS clusters sharing the same VPC ?"
date:   2023-08-03  11:55:00 +1000
categories: aws
image: /assets/images/multi-cluster-eks-vpc.png
slug: eks clusters sharing same vpc
tags: aws eks cluster-upgrades mutiple-cidr-blocks
---

Kubernetes upgrades always have been a risk item in anyones change log , you would have alredy heard enough stop the world stories . One of our customers when upgrading EKS cluster to newer versions wanted to completely avoid a down time scenario due to changes which might break the cluster . The customer was a fintech trading organisation where users of their app span globally and continously use their apps and api`s throughout the day ,which means they could not risk a downtime whatsoever . Transactions amounting to millions are being executed via micros-services each day, a little downtime cost them dearly.

When it comes to Kube some upgrades are straight forward but some are not due to various reasons. For instance , if you have various other components you rely on like service meshes might not work or fail due to version mis-match issues. Also some k8 upgrades might introduce deprecations of the previous api versions and the add-ons and the plugins rely on those api versions might fail to work after the upgrade . Upgrades are envitable , we can't simply ignore it , thats how we keep the cluster upto date with the latest standards and apis versions and security fixes . 

In this case , EKS vpc had three CIDR ranges attached to it where the managed/self managed nodes primary interfaces will reside and the rest of the CIDR ranges were dedicated to the pods . This would allow them to have thousands of pods to be operated having its own IP from the a dedicated VPC range. Public ALB was fronting all the microservices in operation and it was managed seperately not via the kubernetes ingress controller ( explain to you later why this is important in the paticular instance )

The main database serving the stateless microservices resides in the same VPC and also few other services such as AWS managed kafka service and few stateless services some are app specific ,rest of them were Kubernetes Add ons which require stateless deployments ( prometheus , elastisearch so on ). Mongo DB atlas services was used via private link extended from its source VPC.

> How would you Upgrade EKS cluster with least possible risk of data loss or downtime of the services ? 

If the next step up version does not introduce breaking changes neither does your other control plane add-ons such as service meshes , also neither of them introduces changes which distrupts the combatibility matrices then you could easily upgrade them without any worries after initial tests in the pre-prod environments . 

If you are not so lucky then you would have to have to rely on a blue-green strategy rolling out the deployment in a completely new EKS cluster . EKS best practices always suggest you to have cluster per VPC . However , caution should be excercised , if you have statefull deployments and the corresponding persistent volumes are required to be available for the new cluster to readily used by the applications . In the other words , you don't want to put up with taking snapshots of those volumes to be transferred over to the new VPCs or accounts ,but rather want them to be used hot out of the oven .  

The again , you might have DB services tighly coupled to the VPC in which the EKS cluster was deployed to and you would rather not punch holes into the VPC to allow another VPC talking to it . If these are uncompromisable then you have no option than creating another EKS cluster in the same VPC . 

Rolling out a new Green EKS cluster within the same VPC should not be considered as the first approach , only if you have no other option . Even then , you would have to carefully plan your VPC CIDR ranges and subnets does not interfere with your neighbouring cluster. If you are hosting thousands of pods then its better you associate multiple IPv4 CIDR blocks or IPv6 CIDR blocks allowing much room for your neighbouring cluster and its deployments. 

Customer already had four IPv4 CIDR blocks attached to the same VPC and which allowed room for the new cluster and pods to take its presence . New cluster and its managed nodes were created in the non interfering CIDR ranges from those of the old cluster .

Following steps were followed in sequence .  

1. Bootstrap the cluster in those new CIDR ranges hirtherto not used . 
2. Roll out the add-ons and prepheral plugins and validate them if all are working as expected.
3. Deployment of the stateless apps ( make sure the CIDR ranges allocated for the green cluster is whitelisted for the DB connetivity )
4. Slowly allow the ingress traffic to the green deployment via the ALB ( ALB fronting the apps should be able to load balance to the green cluster pods as its not managed via the kubernetes ingress controller ) 
> Registering ALB target groups with the new deployments is not explained in detail here , the intention of the blog is you could have mutiple clusters within the same VPC and we approached it in this situation
5. Tricky part are the stateful applications - unlike the stateless applications you have to come to terms to accept a small downtime ( main aim is to reduce the risk of the downtimes to minimal as possible )
> Due to Amazon EBS constraints, we can't deploy statefulsets in both clusters simultaneously for migration. Amazon EBS volumes can only be mounted on one instance at a time and the same volumes will be used for the new deployment instantly .
6. scale down the stateful set applications to zero replicas 
7. Get the list of all PVs you are planning to migrate 
8. Find all the corresponding EBS Volumes associated with it.
9. Snapshot all the EBS Volumes from the previous step ( in case something goes wrong )
10. Patch PersistentVolumes reclaim policy as Retain. This will keep the EBS volume in AWS even if the PV or PVC is accidentally deleted from the cluster.
11. Backup the PV and PVC definitions in a file ( which will be used in the new cluster )
12. Deploy the stateless apps in the new green cluster and also scale them down to zero so nothing is attached to any PVC in the new environment
13. Delete the new PVCs that was created for those stateful deployments. This should in turn delete the empty PV and EBS Volumes created for the new cluster 
14. Apply the PVC definitions saved up on the step 11
15. Grab the uid of the PVC. This is needed when applying the PV to the cluster.
16. Now we can apply the PV. We will additionally set the claimRef to the uid found in the previous step.
17. it is safe to delete the PVC and PV from the old cluster. Because the reclaim policy was previously set to Retain, the EBS Volume will stay active in AWS.
18. Scale statefull apps on the new cluster to its original replica count. It may take a few minutes for the volume to attach to the new node.

steps from 6 to 18 are common to any migration of stateful apps not limited to this scenario detailed here . 

In the end the traffic was fully transferred to the new cluster and the old cluster was taken down when it was confirmed the entire roll out was successful . 
















 