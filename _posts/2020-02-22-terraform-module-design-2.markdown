---
layout: post
title:  "Terraform Module Design and Thoughts: Part 2"
date:   2020-02-15 23:00:32 -0500
categories: terraform iaac concepts
--- 
### Act 2: Writing a Module

Nothing is stopping you from writing a hugely complex module, and orchestrate your entire infrastructure with only two values, but you probably don't want to.

I consider there to be 3 salient points to module writing: ***Flexibility***, ***Simplicity*** and ***Defensive Terraform***.
#### Flexibility

A module should be a small unit, doing a singular, specific task, and is completely bulletproof. A good module could be:

**RDS Instance**
- RDS Instance
- Parameter Group
- Option Group
- Security Group and Rules

There are multiple resources here, all with intricate relationships to ensure the whole thing is secure. But it doesn't step outside its bounds, a precise scope.

This module invokes ***Flexibility*** as it can now be used in multiple application stack use cases!


#### Simplicity

A module should be very simple. No overtly complex dependencies or hacks. Infrastructure as code can be a complex beast. While you will want to push as MUCH complexity into your module, you don't want it to fall over itself. A (real world) example can be something such as:

*We need a private Network Load Balancer (NLB) to be put in front of a private Application Load Balancer (ALB)*

Therefore in our module we need to:
- Create the ALB
- Wait for the ALB to be provisioned two network cards in the VPC
- Scan what those IP Addresses are
- Create the NLB
- Add the IP addresses to the NLB target Group


This is exceptionally complex as Terraform (as of Feb 2020) does not support exposure of the Load Balacer IP addresses, so we *could* scan them out using the terraform data provider and *then* we can scope it down to all ip addresses associated with the ALB and *then* put them in a data array and *then*....

...No, things are going to wrong, we have started to over-engineer something that COULD be done, but SHOULD we...? Just create two Modules and hope that Hashicorp [Gets around to approving that pull request](https://github.com/terraform-providers/terraform-provider-aws/pull/2901).

A module without these complexities invokes our rule of ***Simplicity***. We don't want complex hacks and strange interactions outside the laws of Terraform, so when it's used by our consumers we don't deal with strange bugs.

#### Defensive Terraform

This is about our lifecycles. We want our terraform to be in small, bite-sized pieces so that if something does go the way of the dodo by accident, we don't kill the whole infrastructure in the process.

This practice has been termed *defensive terraform* as per this podcast i was influenced by: <https://packetpushers.net/podcast/full-stack-journey-027-understanding-infrastructure-as-code-and-terraform-with-curt-micol/>. The main part of this Podcast I take away is the fact your Terraform State should be as small and decoupled as possible.

We want to do the same for our modules, the size of our modules should be defensive in nature, so if a new version DOES end up not playing nicely with itself, we only affect a small part of the infrastructure, and a singular module isn't in charge controlling huge amounts of the stack.

A module that doesn't have any kind of hard coupling or dependencies outside of itself can invoke our rule of ***Defensive Terraform**. So when an unexpected bomb goes off, we only lose one small ship in the fleet.


### Act 3: Deploying a Module


### Act 4: Consuming a Module



