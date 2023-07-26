---
layout: post
title:  "Evolution of a NetworkHub"
date:   2023-07-14  16:20:40 +1000
categories: cloud aws
image: multiregional-network-high-tolerance-hub.png
---

In this blog post, we'll talk about how a basic network setup has grown and changed over time. Just from a single VPC connecting to a data centre to a fully fledged multi regional hub and spoke architecture connecting multi regional remote sites .


## Whats the customer initially wanted ? 

The customer needed a solution to connect their remote sites, which were spread out from Australia to as far as Africa and Canada. These sites were quite remote and had limited internet connectivity. Consequently, they had to depend on satellite internet service providers like Starlink to establish connectivity.

what we proposed was a simple VPC connecting all remote sites via site to site VPN. 

> It's always essential to consider as a prerequisite whether your customer gateway peer supports the default behavior of the AWS side of the connection

below figure illustrates the initial design 

### _Initial Design_
![image]({{ site.baseurl }}/assets/images/generic_initial.png)

> Transit Gateway was used to give connectivity between VPCs spread across accounts and the site to site connections terminated at the TGW instead of a virtual private gateway. 

since the remote sites are even further from the home VPC region and they rely on 3G/4G/satelitte internet connectivity , we suggested them to use AWS global accelarator on the VPNs to route traffic through the nearest AWS point of presence. 

with the inclusion of the Global Accelarator the design looked as shown below 


### _With the inclusion of GA_
![image]({{ site.baseurl }}/assets/images/generic_with_ga.png)

Imagine you're a customer using a cloud service for important tasks. Just like anyone else, you wouldn't want all your important services to rely solely on one cloud provider. That would be risky, as it could lead to a single point of failure. So, to avoid that, you'd prefer to distribute the risks by not depending entirely on one cloud vendor.


They sought the help of palo alto ( prisma cloud ) to provide a middle layer between the remote sites and the AWS side and also at the same time use the nearest point of presence provided by palo to alto to stay closer to their remote sites . 

You would have already guessed how the design would be transformed into as you would see soon .


![image]({{ site.baseurl }}/assets/images/network-hub-palo-alto.png)

At this point in the design, we had successfully ensured high availability (meaning it could withstand any potential zonal failures) by using multiple availability zones and avoiding reliance on a single cloud vendor. However, there was still one concern that bothered the customer.

***What if an AWS region or Palo Alto regional construct fails , will it pull down the rest of the services together with it ?***

simple answer is **_yes_** . it would inevitably pull the rest of it down and how would you circumvent it ? 

So, we came to a decision: to make use of AWS regions. This choice greatly enhances the architecture's ability to handle faults and ensures overall stability. In this setup, each region will have its own network hub, just like in the initial design, and we'll maintain the same number of Virtual Private Clouds (VPCs) across AWS Accounts. All these network hubs will be interconnected in a mesh-like pattern.Each Site to site connection originating from Palo Alto Prisma will be terminated at each end of the regional network hubs as shown below .

initially every site to site VPN connection was rolled out to a single networkhub as show below 

### _Redundant Site to Site terminating at a network hub_

![image]({{ site.baseurl }}/assets/images/multi-regional-networkhub.png)

then once the rest of the regional network hubs were ready those redudant connections made its way to its intended regional hub as shown below

### _Site to Site terminating at each regional hub_

![image]({{ site.baseurl }}/assets/images/multiregional-network-high-tolerance-hub.png)


## Implementation 


[AWS serverless network orchestrator](https://github.com/aws-solutions/network-orchestration-for-aws-transit-gateway) was chosen as the automation tool for the multi regional network hub implementation. The tool was deployed using the [customisations for control tower](https://github.com/aws-solutions/aws-control-tower-customizations) from the master account .The orchestrator was seamless in its execution for applying attachments and associations against the TGW ( transit gateway) and its route tables. Even the peering between regional TGW were applied using the orchestrator. 

> As we were setting up the orchestrator on a brand-new AWS region, we encountered a situation where certain services were not yet accessible. This prevented us from fully utilizing the orchestrator's complete range of features. To work around this, we made a choice to programmatically exclude those services by making changes to the code.This way, we could still proceed with the implementation without those specific features.

Due to security and cost optimisation considerations an egress VPC for each regional network-hub was introduced to direct all the outbound internet traffic from the spoke VPCs to go through it. In addition, interface endpoints with its own private PHZ were implemented on the same egress VPC to be share across the network hub

## Conclusion

In simpler terms, We added an extra layer of protection to their cloud setup by bringing on another cloud provider between regional sites to AWS and also by opting in for AWS regional redundancy by way of multi regional networkhub , making it more reliable,fault tolerant and redundant. This way, if one part of the system fails, it doesn't bring down everything else with it. It's like having a backup plan for when things don't go as expected. This design transformation helps ensure a smoother and safer cloud experience overall.




