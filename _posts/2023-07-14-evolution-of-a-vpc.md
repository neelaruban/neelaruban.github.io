---
layout: post
title:  "Evolution of a NetworkHub"
date:   2023-07-14  16:20:40 +1000
categories: cloud aws
image: multiregional-network-high-tolerance-hub.png
---

this blog post discusses the evolution of a network architecture from a single VPC  connecting to a data centre to a fully fledged multi regional hub and spoke architecture connecting regional remote sites .


## Whats the customer initially wanted ? 

The customer wanted a way to connect to their remote sites spanning from australia to as far africa and canada. Those remote sites were literally remote without much internet connectivity as such they had to rely on satellite internet services vendors such as starlink to provide connectivity . 

what we proposed was a simple VPC using site to site VPN connecting all remote sites . 

> its always good to consider as a prerequiste if the AWS or any cloud vendor for that matter   the default behaviour of the aws side of the connection supported by the remote peer customer gateway 

below figure illustrates the initial design 

### _Initial Design_
![image]({{ site.baseurl }}/assets/images/generic_initial.png)

> Transit Gateway was used to give connectivity between VPCs spanning accounts and the site to site connections terminated at the TGW instead of a virtual private gateway. 

since the remote sites are even further from the home VPC region and they rely on 3G/4G/satelitte internet connectivity , we suggested them to use AWS global accelarator on the VPNs to route traffic through the nearest AWS point of presence. 

with the inclusion of the Global Accelarator the design looked liked below 


### _With the inclusion of GA_
![image]({{ site.baseurl }}/assets/images/generic_with_ga.png)

Imagine you're a customer using a cloud service for important tasks. Just like anyone else, you wouldn't want all your important services to rely solely on one cloud provider. That would be risky, as it could lead to a single point of failure. So, to avoid that, you'd prefer to distribute the risks by not depending entirely on one cloud vendor.


They sought the help of palo alto ( prisma cloud ) to provide a middle layer between the remote sites and the AWS side and also at the same time use the nearest point of presence provided by palo to alto closer to their remote sites . 

You would have already guessed how the design would be transformed into as you would see soon as depicted below 


![image]({{ site.baseurl }}/assets/images/network-hub-palo-alto.png)

At this point in the design, we had successfully ensured high availability (meaning it could withstand any potential zonal failures) by using multiple zones and avoiding reliance on a single cloud vendor. However, there was still one concern that bothered the customer.

***What if an AWS region or Palo Alto regional construct fails , will it pull down the rest of the services together with it ?***

simple answer is **_yes_** . it would inevitably pull the rest of it down and how would you circumvent it ? 

So, we came to a decision: to make use of AWS regions. This choice greatly enhances the architecture's ability to handle faults and ensures overall stability. In this setup, each region will have its own network hub, just like in the initial design, and we'll maintain the same number of Virtual Private Clouds (VPCs) across AWS Accounts. All these network hubs will be interconnected in a mesh-like pattern.Each Site to site connection originating from Palo Alto Prisma will be terminated at each of the regional network hubs as shown below .

initially every site to site VPN connection was rolled out to a single networkhub as show below 

### _Redundant Site to Site terminating at a network hub_

![image]({{ site.baseurl }}/assets/images/multi-regional-networkhub.png)

then once the rest of the regional network hubs were ready those redudant connections made its way to its intended regional hub as shown below

### _Site to Site terminating at each regional hub_

![image]({{ site.baseurl }}/assets/images/multiregional-network-high-tolerance-hub.png)


## Implementation 


[AWS serverless network orchestrator](https://github.com/aws-solutions/network-orchestration-for-aws-transit-gateway) was chosen as the automation tool for the multi regional network hub implementation. The tool was deployed using the [customisations for control tower](https://github.com/aws-solutions/aws-control-tower-customizations) pipeline via the master account .The orchestrator was seamless in its execution for applying attachments and associations for TGW and its route tables. Even the peering between regional TGW were applied using the orchestrator. 

> During the implementation of the orchestrator on a newly minted AWS region there were some services which were not yet available to fully leverage the featureset of the orchestrator. To work around this, we made a choice to programmatically exclude those services by making changes to the code.This way, we could still proceed with the implementation without those specific features.

Due to security and cost optimisation considerations an egress VPC for each regional network-hub was introduced to direct all the outbound internet traffic from the all the spoke VPCs to go through it. In addition, interface endpoints with its own private PHZ were implemented on the same egress VPCs to keep the Spoke VPC traffic towards those AWS services within the AWS backbone.

## Conclusion

In simpler terms, We added an extra layer of protection to their cloud setup by bringing on another cloud provider between regional sites to AWS and also by opting in for AWS regional redundancy by way of multi regional networkhub , making it more reliable,fault tolerant and secure. This way, if one part of the system fails, it doesn't bring down everything else with it. It's like having a backup plan for when things don't go as expected. This design transformation helps ensure a smoother and safer cloud experience overall.




