---
layout: post
title:  "Terraform Module Design and Thoughts"
date:   2020-02-15 23:00:32 -0500
categories: terraform iaac concepts
---
I spend a good chunk of my day not so much deploying infrastructure, but creating the structures that go around the best ways to
deploy said infrastructure. The types of people I encounter are also what i'd consider to be very smart, resourceful people but need a way to tap into the knowledge of people from the past, who have travelled the roads, built the bridges and created the post offices.

From this, comes a need to codify our learnings into a repeatable pattern. Infrastructure as Code is the big driver behind that enables me to pass on the knowledge of how my bridge over that river, or how we were able to connect our two post offices together.

My main tool of choice over the years has been Terraform . I enjoy the heavy focus on modularity and treating these module structures as artifacts themselves. However i've been a user of both Cloudformation from AWS and ARM from Azure, which can also support similar concepts (albeit, in the CFN world its a little less 'modular' and more 'multi decoupled stack'). I may write about these as well in the future...

Over a few posts I want to go over a few of my ideas and opinions i've formed on codifying IaaC for general users at all cloud skill levels to consume. They are:

- Anatomy and the 'Rules' of a good Module
- Deploying a Module
- Consuming a Module
- Closing Thoughts.

But before all that...

### Act 1: What even is a terraform module?
The first thing to grasp is where a module lives in grand scheme of Terraform; [In a short sentence I stole from Hashicorp:](https://www.terraform.io/docs/modules/index.html)

*A module is a container for multiple resources that are used together. Modules can be used to create lightweight abstractions, so that you can describe your infrastructure in terms of its architecture, rather than directly in terms of physical objects.*

So, what does this mean for our intrepid module builders and their consumers...


If we consider a module to be a collection of these AWS Resources:
- S3 Bucket
- EC2 Instance
- EC2 Role
- Security Group

And the module's readme has stated: *This Module will spin up an EC2 Instance, allowing for SSH Access from a single IP and an S3 bucket that the Instance can Read and Write to.*


The consumer may only be asked to provide values for terraform such as:
- *What name should the S3 Bucket and EC2 instance have?*
- *What S3 Operations should the EC2 instance need to do to the bucket? Read/Write or just Read?*

The module author has wrote the module in such a way that he has abstracted the complexity away. Our consumer only needs to answer the questions above to save him from 'S3 Bucket Leak of the Week'.


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




