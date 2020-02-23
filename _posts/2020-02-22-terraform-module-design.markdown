---
layout: post
title:  "Terraform Module Design - Part 1"
date:   2020-02-15 23:00:32 -0500
categories: terraform iaac concepts
---
I spend a good chunk of my day not so much deploying infrastructure, but creating the structures that go around the best ways to
deploy said infrastructure. The types of people I encounter are also what i'd consider to be very smart, resourceful people but need a way to tap into the knowledge of people from the past, who have travelled the roads, built the bridges and created the post offices.

From this, comes a need to codify our learnings into a repeatable pattern. Infrastructure as Code is the big driver behind that enables me to pass on the knowledge of how my bridge got over that river, or how we were able to connect our two post offices together.

My main tool of choice over the years has been Terraform . I enjoy the heavy focus on modularity and treating these module structures as artifacts themselves. However i've been a user of both Cloudformation from AWS and ARM from Azure, which can also support similar concepts (albeit, in the CFN world its a little less 'modular' and more 'multi decoupled stack'). I may write about these as well in the future...

Over a few posts I want to go over a few of my ideas and opinions i've formed on codifying IaaC for general users at all cloud skill levels to consume. They are:

- Anatomy and the 'Rules' of a good Module
- Deploying a Module
- Consuming a Module
- Closing Thoughts.

But before all that...

## Act 1: What even is a "Terraform Module"?
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
- *What IP address can i SSH From?*

The module author has wrote the module in such a way that he has abstracted the complexity away, the module author has included all the security controls we would expect from the resources used to create this stack. 

Our module consumer only needs to answer the questions above to take advantage of this, and save himself from maybe accidentally spinning up a wide open S3 bucket, giving himself a shiny 'Insecure S3 Bucket of the Week' award that you see every few days in [*The Register*](https://www.theregister.co.uk/).



As I go through the above subjects, i'll also be writing a set of modules using some of the tenets and mantra's ive developed for myself through this. So Code snippets and real life examples will be available for use



