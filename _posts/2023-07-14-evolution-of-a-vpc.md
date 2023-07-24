---
layout: post
title:  "Evolution of a NetworkHub"
date:   2023-07-14  16:20:40 +1000
categories: cloud aws
image: multi-regional-networkhub.png
---

this is posts discusses the evolution of a network architecture from a single VPC just connecting to the data centre to a fully fledged multi regional hub and spoke architecture connecting regional remote sites .

## following key elements will be covered 

> this is currently work in progress document

- whats the customer initially wanted ? 
- what we proposed 
- various designs which came out of the workshops 

- what we stuck to ? 
- key parameters influenced the decisions 
- how the implementation was done to support multi regional deployment - network orchestrator 

- best practices for network orchestrator - 
- specifically around maitenance of the front end apps and how to avoid them altogeth 
- how we got around some of the services not being availabile at the time of the implementation in some of the regions.

- automation of the creation of VPC end points and sharing their private hosted zones accross multi VPC set across accounts 

### Some of the Architecture Diagrams

***

#### Initial Design

![image]({{ site.baseurl }}/assets/images/generic_initial.png)

#### Revision

![image]({{ site.baseurl }}/assets/images/generic_with_ga.png)

### Revision II

![image]({{ site.baseurl }}/assets/images/network-hub-palo-alto.png)

### Revision III

![image]({{ site.baseurl }}/assets/images/multi-regional-networkhub.png)