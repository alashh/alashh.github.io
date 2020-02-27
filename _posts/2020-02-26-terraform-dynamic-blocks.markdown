---
layout: post
title:  "Terraform Dynamic Blocks"
date:   2020-02-26 21:43:32 -0500
categories: terraform iaac concepts
---

Fielded a question from a colleague today on an issue he was having related to our Terraform S3 Bucket Module. We had a requirement to support [lifecycle Policies](https://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html), however we needed to support an indeterminate amount of these policies.


## The Ask
See, the way Terraform deals with these is as follows, per the [Terraform S3 Bucket Lifecycle Documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html):

```
resource "aws_s3_bucket" "bucket" {
  bucket = "my-bucket"
  acl    = "private"
  lifecycle_rule {
    id      = "log"
    enabled = true

    prefix = "log/"

    tags = {
      "rule"      = "log"
      "autoclean" = "true"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA" # or "ONEZONE_IA"
    }

    transition {
      days          = 60
      storage_class = "GLACIER"
    }

    expiration {
      days = 90
    }
  }
}
```
Of course, we could have 1, 10, 100 of these blocks in here, how can we dynamically add these in...

...In previous experiences in earlier Terraform, it REALLY hates iteration over blocks. You'd run into issues with list positioning being immutable. 

Strange things would happen, like:

## The Dark Past
If you say, wanted four of these above lifecycle rules, you may find a way to hack in a count of some sort over the top of them (I've done something similar before, but cannot recall how I got myself into this mess...)

- We have our iteration; "0, 1, 2, 3" sets of rules
- Lets remove rule 2...
- ... Now rule 3 is 2, and Terraform compares this to the state
- Its now 'deleting' 3 and 'changing' 2. But we only deleted 2

**But all we wanted to do was just remove 2!!!!**

## Terraform Dynamic Blocks and for_each

Somehow, with a bit of luck and google i came across [Dynamic Blocks](https://www.terraform.io/docs/configuration/expressions.html#dynamic-blocks). I have no idea how I missed this before!

The idea behind this little beauty is you can write a `dynamic "settingname" {}` block in replacement of what could be a normal block of parameters for a certain resource. You may want to read Terraform harp on about though because a professional probably wrote that. I'm just going to use examples.

So in our S3 case rather than:
```
resource "aws_s3_bucket" "bucket" {
  bucket = "my-bucket"
  acl    = "private"
  lifecycle_rule {
    id      = "log"
    enabled = true
  }
}
```

We instead could write something like:
```
variable "lifecycle_rule" {
  type        = list(map(string))
  description = "Map of Lifecycle Rules"
}

resource "aws_s3_bucket" "bucket" {
  bucket = "my-bucket"
  acl    = "private"

  dynamic "lifecycle_rule" {
    for each = var.lifecycle_rules
    content {
        id      = ingress.value["id"]
        enabled = ingress.value["enabled"]
    }
  }
}
```
<sub>code blocks shortened for brevity, these don't actually work<sub>

- Denote here the `dynamic` in front of where the block would normally start, and lifecycle_rule in the section where a value would go
- This tells us to treat this block as something we can iterate over
- We then need a list of maps to iterate over for our values, for this we are using `var.lifecycle_rules`

So, what could lifecycle_rules look like?
```
  lifecycle_rules = [
    {
      id = "1"
      value = "true"
    },
    {
      id = "2"
      value = "false"
    },
```

Great! True Iteration... Now let's apply it to something.

## Making a Dynamic Resource Module

Instantly, I jumped in and wrote a pretty basic proof of concept using an AWS Security Group and Ingress Rules. I didn't write the S3 bucket with lifecycles due to the fact there are maps inside maps (among other things) so I need to have a bit more of a think on this. For now let's keep it VERY simple.

**You can find my code at: <https://github.com/adamlash/tf-aws-securitygroup-dynamic> but don't laugh too hard.**


### The Resource
in the *main.tf* we have:
```
resource "aws_security_group" "default" {
  name = var.name
  description    = var.description
  vpc_id  = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_cidrs
    content {
      from_port = ingress.value["from_port"]
      to_port   = ingress.value["to_port"]
      protocol  = ingress.value["protocol"]
      cidr_blocks = [ingress.value["cidr_blocks"]]
    }
  }
  tags = var.tags
}
```

Pretty basic Security Group here. The main change here is out `dynamic "ingress"{}` block and the values from the list. 

now lets build our *variables.tf* file.
```
...
variable "ingress_cidrs" {
  type        = list(map(string))
  description = "Map of Ingress CIDR config"
}
...
```

I'm only showing the ingress_cidrs var here, the rest are boring strings. But from there we are expecting a *list of maps*. So we should provide our module a collection of the to/from port, protocol and cidr_blocks

so, lets check out the *examples/singlegroup.tf*. A file with the module being referenced
```
module "aws_dynamic_sg" {
  source = "../"
  name = "al-dynamic-sg1"
  description    = "testing dynamic rules"
  vpc_id = "vpc-0a027487e9e5ebf21"
  ingress_cidrs = [
    {
      from_port = "3389"
      to_port = "3389"
      protocol = "tcp"
      cidr_blocks = "0.0.0.0/0"
    },
    {
      from_port = "444"
      to_port = "444"
      protocol = "tcp"
      cidr_blocks = "0.0.0.0/0"
    },
  ]
}
```

And in here, we have our `ingress_cidrs` list of maps. We should expect this to create a Security Group with 3389 and 444 open from 0.0.0.0/0, so lets run Terraform


## Standing up the Module, and trying to break it.

#### First run:
```
  # module.aws_dynamic_sg.aws_security_group.default will be created
  + resource "aws_security_group" "default" {
      + arn                    = (known after apply)
      + description            = "testing dynamic rules"
      + egress                 = (known after apply)
      + id                     = (known after apply)
      + ingress                = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 3389
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 3389
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 444
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 444
            },
        ]
      + name                   = "al-dynamic-sg1"
      + owner_id               = (known after apply)
      + revoke_rules_on_delete = false
      + vpc_id                 = "vpc-0a027487e9e5ebf21"
    }
```
<sub>going forward i'll be chopping the non-ingress chaff out of the clips<sub>

We are creating our SG, with two Ingress rules, brilliant! Lets Apply that, and remove one.

#### Second Run, Removing 3389:
```
      ~ ingress                = [
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - description      = ""
              - from_port        = 3389
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "tcp"
              - security_groups  = []
              - self             = false
              - to_port          = 3389
            },
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - description      = ""
              - from_port        = 444
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "tcp"
              - security_groups  = []
              - self             = false
              - to_port          = 444
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = null
              + from_port        = 444
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 444
            },
        ]
```

Hmm, so denote that it **Deleted, then re-created 444**. So it looks like the tfstate still has some kind of immutable order of rules behind the scenes (or, at least for Security Groups), lets continue messing with it though.

#### Third run: change 444 to 80801
```
      ~ ingress                = [
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - description      = ""
              - from_port        = 444
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "tcp"
              - security_groups  = []
              - self             = false
              - to_port          = 444
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 8081
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 8081
            },
```

Replacing a value, as expected deleted the old one and re-created the new one.

#### Fourth run: Deleting an old value and adding a new one:
```
      ~ ingress                = [
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - description      = ""
              - from_port        = 8081
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "tcp"
              - security_groups  = []
              - self             = false
              - to_port          = 8081
            },
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = ""
              + from_port        = 444
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 444
            },
```

As expected, same behaviour as changing the value, the re-creation is expected.



## Findings and Future Sleuthing

So, it seems like a HUGE thing to investigate when it comes to modules that have blocks of values we cannot determine at time of creation, for things such as Security Groups and Lifecycle Policies... *however* the fact the list SEEMS to held in some kind of order back over in the tfstate is interesting, more investigation is needed.

So, what do I plan to do next?

- Figure out how to use this for S3 Lifecycle Policies.
- Figure out whats actually happening in the statefile, is it just flattening it out or does it know of the Dynamic List?
- Find out more ways to query the map, i've seen some nontrivial modules on github [use count and lookups in for_each and their values respectively](https://github.com/operatehappy/terraform-aws-s3-bucket/blob/master/main.tf).

Overall, i feel pretty good on this. Can't wait to mess with it more.

### Appendix and Links
**My Source Code**

<https://github.com/adamlash/tf-aws-securitygroup-dynamic>

**Terraform Dynamic Blocks**

<https://www.terraform.io/docs/configuration/expressions.html#dynamic-blocks>

**Terraform Hello World**

<https://github.com/hashicorp/terraform-guides/tree/master/infrastructure-as-code/terraform-0.12-examples/advanced-dynamic-blocks>


**Nontrivial Example of an S3 Bucket**

<https://github.com/operatehappy/terraform-aws-s3-bucket>