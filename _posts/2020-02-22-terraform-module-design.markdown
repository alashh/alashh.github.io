---
layout: post
title:  "Terraform Module Design and Thoughts"
date:   2020-02-15 23:00:32 -0500
categories: jekyll update
---
I spend a good chunk of my day not so much deploying infrastructure, but creating the structures that go around the best ways to
deploy said infrastructure. The types of people I encounter are also what i'd consider to be very smart, resourceful people but need a way to tap into the knowledge of people from the past, who have travelled the roads, built the bridges and created the post offices.

From this, comes a need to codify our learnings into a repeatable pattern. Infrastructure as Code is the big driver behind that enables me to pass on the knowledge of how my bridge over that river, or how we were able to connect our two post offices together.

My main tool of choice over the years has been Terraform . I enjoy the heavy focus on modularity and treating these module structures as artifacts themselves. However i've been a user of both Cloudformation from AWS and ARM from Azure, which can also support similar concepts (albeit, in the CFN world its a little less 'modular' and more 'multi decoupled stack'). I may write about these as well in the future...

Either way, module considerations, design and structure!

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

You could write your entire stack in a single module. But there are three main points to consider with modules

#### Flexibility

A module should be a small unit, doing a singular task very damn well. For example:

##### My S3 Bucket Module
Resources:
- RDS Instance
- Security Group




#### Simplicity

#### Defensive Terraform



