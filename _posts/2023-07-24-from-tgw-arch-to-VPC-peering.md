---
layout: post
title:  "VPC Peering is still your friend"
date:   2023-07-24  17:27:11 +1000
categories: aws
image: vpc-peering.png
---

I once worked with a customer who had an interesting networking topology. They had a multi VPC hub and spoke network with around forty VPCs connected to a TGW (Transit Gateway) . The purpose of the TGW was to connect to a specific VPC that held the control plane, which managed the data plane in the other VPCs. 

Only the control plane VPC should be able to connect to the data planes in each VPCs but there should not be any connectivity between data planes as each data plane VPC is customer specific part of a multi customer SAAS model . Further, each of the data plane VPC is connected to another transit gateway from the customer side, which we cover later.In essence, it forms a sort of maze-like structure.

to illustrate my description let me give you the architecture diagram of it 

![image]({{ site.baseurl }}/assets/images/vpc-peering-topology.png)

TGW automatically comes with a default route table. By default, this route table is the default association route table and the default propagation route table. Unfortunately the Customer has not opted out of this setting resulting in any VPCs attached to the TGW gaining access to each of the VPCs attached to it. 

This is big red flag enabling any customer side VPC gaining a free pass to rest of the customer side VPCs . 

include a gif depicting a massive blow 

Another important thing to notice here is that any customer side data plane VPC could act as a *_ bridge VPC_* between the main TGW ( TGW A from the picture ) and the customer side of the TGW ( TGW B from the picture ). 

this could only be possible when there is a default static route table entry pointing to a data plane VPC attachment from both Transit gateways, then a again its just a mistake a away from being one. A good architectural design should not leave room for any mistakes happening in the future.

This interesting topology where a VPC acting as a bridge between two transit gateways is being discussed in detail in this  [blog](https://www.gilles.cloud/2019/12/aws-transitive-routing-with-transit.html) post from Giles. This architecture was quite sought after if someone wanted to peer TGW from the same region , until recently it was only possible via a bridge VPC .Now the intra regional peering is supported out of box. 



***How could we avert any possibility of mistakes happening in the future , meaning how to achieve fail proof ?***

practically , complete fail proof is impossible but we could reduce the exposure to the downside to the minimal . 

There is always a simple solution for a complex problem such as this , bring on our old friend *VPC peering*

VPC peering is an ideal solution for a problem statement such as this , which is as follows  

## Problem Statement

1. Centralised control plane communicating bi directionaly to the data planes. 
2. In this case data planes should not strictly talk to each other because they belong to different perimeters.
3. Any new introduction of a data plane should easily scale and provide connectivity between both effortlessly. 
4. Data plane instances should be able to limit the ingress traffic to nothing but only from the control plane .

Besides , not so very recently AWS announced all data transfer over a VPC Peering connection that stays within an Availability Zone (AZ) are going to be free. Which is a great news for VPC peering users as TGW is an expensive service as means to the same end . 

Only constraint is that you could have peering connections upto 50 for a VPC and could adjusted upto 125 connections but it still gives you the ability to work with 125 different data plane VPCs ( as many different customers in this case )

if the company is scaling too fast and if they feel like they going to exhaust the limits soon they could certainly have a new control plane VPC and as many data plane VPCs. This gives them redundancy and some sort of partioning of their control plane with little to no marginal cost. 

> Transit gateway costs comes in two parts - TGW attachment costs ( billed hourly ) , data transfer costs be between same AZ or inter AZ . Peering does not carry any hourly attachment costs as such .

Peering is also ideal if you want to reference the source security groups from the peered VPC for inbound and outbound security group rules . The peered VPC could be in same account or from a different account sounds great isn't :) 



The main lesson here is not to be swayed by shiny new services just because they sound exciting and others are using them. Instead of succumbing to FOMO (Fear Of Missing Out), we should approach things from first principles and ask ourselves if the existing solutions already solve the problem effectively. Many times, I've found that the existing solutions are actually ideal for common problems like these.