---
layout: post
title:  "Could we have two EKS clusters sharing the same VPC ?"
date:   2023-08-03  11:55:00 +1000
categories: aws
image: /assets/images/multi-cluster-eks-vpc.png
slug: eks clusters sharing same vpc
tags: aws eks cluster-upgrades mutiple-cidr-blocks
---

Kubernetes upgrades have always been a risk item in anyone's change log. You would have already heard enough "stop the world" stories. One of our customers, when upgrading their EKS cluster to newer versions, wanted to completely avoid a downtime scenario due to changes that might break the cluster. 

The customer was a financial technology trading company. People all around the world use their apps and tools all the time meaning they could not risk any downtime whatsoever. Transactions amounting to millions are being executed via microservices each day, and a little downtime costs them dearly.

When it comes to Kubernetes, some upgrades are straightforward, but some are not due to various reasons. For instance, if you have peripheral components you rely on, like service meshes, they might not work or could fail due to version mismatch issues. Also, some Kubernetes upgrades might introduce deprecations of the previous API versions, and the add-ons and plugins that rely on those API versions might fail to work after the upgrade. Upgrades are inevitable; we can't simply ignore them; that's how we keep the cluster up to date with the latest standards, API versions, and security fixes.

In this case, the EKS VPC had three CIDR ranges attached to it, where the managed/self-managed nodes' primary interfaces would reside, and the rest of the CIDR ranges were dedicated to the pods. This allowed them to operate thousands of pods, each having its own IP from a dedicated VPC range. A public ALB was fronting all the microservices in operation, and it was managed separately, not via the Kubernetes ingress controller (explained to you later why this is important in the particular instance).

### _Initial Design_

The apps managed by the cluster are fronted by an ALB which was separately managed not through the ingress controller which proved to be very good decision to decouple them early on as the same ALB will be used to front the Apps orchestrated by the Cluster 2 within the same VPC .Nowadays , Gateway controller could be used to do the same thing despite having controllers on both clusters talking over the ALB.  

![image]({{ site.baseurl }}/assets/images/wazirx-design.svg)

As a precaution we had to take a full backup of the existing cluster using velero , since we don't want to take a risk of losing out on important data for the stateful apps in the event of total failure . Its always a good practices to expect failures and plan it accordingly . 

### _Velero Backup_
![image]({{ site.baseurl }}/assets/images/velero-bk.svg)

***How would you Upgrade EKS cluster with the least possible risk of data loss or downtime of the services?***

If the next step-up version does not introduce breaking changes, nor do your other control plane add-ons, such as service meshes, and if none of them disrupts the compatibility matrices, then you could easily upgrade them without any worries after initial tests in the pre-prod environments.

If you are not so lucky, then you would have to rely on a blue-green strategy, rolling out the deployment in a completely new EKS cluster. EKS best practices always suggest having a cluster per VPC. However, caution should be exercised if you have stateful deployments and the corresponding persistent volumes are required to be available for the new cluster to be readily used by the applications. In other words, you don't want to deal with taking snapshots of those volumes to be transferred over to the new VPCs or accounts, but rather want them to be used hot out of the oven.

Then again, you might have DB services tightly coupled to the VPC in which the EKS cluster was deployed, and you would rather not punch holes into the VPC to allow another VPC to talk to it. If these are uncompromisable, then you have no option but to create another EKS cluster in the same VPC.

Rolling out a new Green EKS cluster within the same VPC should not be considered as the first approach, only if you have no other option. Even then, you would have to carefully plan your VPC CIDR ranges and subnets not to interfere with your neighboring cluster. If you are hosting thousands of pods, then it's better to associate multiple IPv4 CIDR blocks or IPv6 CIDR blocks, allowing much room for your neighboring cluster and its deployments.

The customer already had four IPv4 CIDR blocks attached to the same VPC, which allowed room for the new cluster and pods to take its presence. The new cluster and its managed nodes were created in the non-interfering CIDR ranges from those of the old cluster.

Following steps were followed in sequence.

1. Bootstrap the cluster in those new CIDR ranges attached to the VPC hitherto not used.
2. Roll out the add-ons and peripheral plugins and validate them if all are working as expected.
3. Deployment of the stateless apps (make sure the CIDR ranges allocated for the green cluster are whitelisted for the DB connectivity)
4. Slowly allow the ingress traffic to the green deployment via the ALB (ALB fronting the apps should be able to load balance to the green cluster pods as it's not managed via the Kubernetes ingress controller)
> Registering ALB target groups with the new deployments is not explained in detail here; the intention of the blog is you could have multiple clusters within the same VPC, and we approached it in this situation.
5. The tricky part is the stateful applications - unlike the stateless applications, you have to come to terms with accepting a small downtime (the main aim is to reduce the risk of the downtimes to a minimum as possible).
> Due to Amazon EBS constraints, we can't deploy stateful sets in both clusters simultaneously for migration. Amazon EBS volumes can only be mounted on one instance at a time, and the same volumes will be used for the new deployment instantly.
6. Scale down the stateful set applications to zero replicas.
7. Get the list of all Persistent Volumes ( PV) you are planning to migrate.
8. Find all the corresponding EBS Volumes associated with them.
9. Snapshot all the EBS Volumes from the previous step (in case something goes wrong).
10. Patch PersistentVolumes reclaim policy as Retain. This will keep the EBS volume in AWS even if the PV or Persistent Volume Claims ( PVC ) is accidentally deleted from the cluster.
11. Backup the PV and PVC definitions in a file (which will be used in the new cluster).
12. Deploy the stateless apps in the new green cluster and also scale them down to zero so nothing is attached to any PVC in the new environment.
13. Delete the new PVCs that were created for those stateful deployments. This should, in turn, delete the empty PV and EBS Volumes created for the new cluster.
14. Apply the PVC definitions saved up on step 11.
15. Grab the UID of the PVC. This is needed when applying the PV to the cluster.
16. Now we can apply the PV. We will additionally set the claimRef to the UID found in the previous step.
17. It is safe to delete the PVC and PV from the old cluster. Because the reclaim policy was previously set to Retain, the EBS Volume will stay active in AWS.
18. Scale stateful apps on the new cluster to its original replica count. It may take a few minutes for the volume to attach to the new node.

Steps from 6 to 18 are common to any migration of stateful apps, not limited to this scenario detailed here.

> important thing to keep in mind when reusing retained PVs is deploying a new PV with the same name would not work as in deploying a new release with `PVC.spec.volumeName`. This will open up anyone to hijack the PVs by just knowings its name. Hence an admin intervention is required to delete the `PV.Spec.ClaimRef` from the retained PVs or pre-fill `PV.Spec.ClaimRef` with a pointer to the new PVC as explained [here](https://github.com/kubernetes/kubernetes/issues/48609#issuecomment-314066616)

In the end, the traffic was fully transferred to the new cluster, and the old cluster was taken down when it was confirmed that the entire rollout was successful.







 