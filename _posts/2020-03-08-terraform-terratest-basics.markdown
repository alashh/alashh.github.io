---
layout: post
title:  "Testing Terraform with Terratest"
date:   2020-03-03 14:43:32 -0500
categories: blog terraform
tags: terraform iaac concepts terratest go
---
A while back I was on the hunt for messing with Testing Frameworks for infrastructure. The use-case at the time was confirming if I could do some kind of nightly 'build' on AWS AMI's, spin it up and confirm the AMI was within spec. From this thoughts I was introduced into the messy world of testing infrastructure.

There's a few players here. I've always wanted to explore them all however not really had to chance in my day to day due to restrictions on my current CI/CD flows at my employer. However with everything starting to fall into a little more place, i've now had a bit of a need to get some off this running. I've started off the journey with [Terratest](https://terratest.gruntwork.io/). It's not AMI testing, but we are in the same ball park.

## What are we trying to achieve here?
So, as in most (modern) development these days, everyone loves to employ TDD - Test Driven Development... or at least maybe we like to think we do. If you don't know what it is, that's okay. Asking Google will tell you more than you need to know on the subject, but the general idea behind it is **writing a test before the code to fail before writing functional code**. The test in this case defines the *end target state* and we build our code to fit this target state. 

So, in our usual day-to-day writing a Terrform module we may

1. Write up our Main.tf, Outputs, Variables etc.
2. Run it, check the results
3. GOTO 1, fix up the tf files and so on
4. Check results again.


However, if we wanted to employ TDD in Terraform:

1. Write a test specifying what we want our Infrastructure to look like
2. Run the test, watch it fail
3. Write up our Main.tf, Outputs, Variables etc.
4. Run the test, did we meet our target state?
5. GOTO 3 if target state is not desirable. If it is, great!


Of course, we say *test* but what we actually writing our tests in? For that fact how can we even **TEST** infrastructure? Normally in software development our *unit tests* would be on the machine itself. `Does 1+1=2? If so - Pass`. We can't really mock AWS and Azure on our local machine (well, at least in the most like-for-like way anyway).


## How do we even Test? What do we even use to test?

This is when our boy [Terratest](https://terratest.gruntwork.io/) comes into play. Terratest works in a pretty basic way. We can write our tests in Golang, asking what input values, regions and general config we want to give to a module. Once we do that we can provide assertions as to what our infrastructure should like.

Now that's been done, we can run Terratest and it will Spin up our infrastructure with the value we provided, then assert against what we expected it to look like. Alright, our missing link is here. Even though its not a true local test like our unit test brothers have, it's definitely something we can work with to confirm our modules are within spec.


## Writing the test

So, i've wrote a quick little Module and Test for S3. A little cribbed from terratest but lets keep it simple for now. I want to iterate on this over the document.

The Git Repo can be found at: [tf-aws-s3bucket](https://github.com/adamlash/tf-aws-s3bucket)

In here, we have our module, a boring S3 bucket:

### The Module
#### main.tf
```
resource "aws_s3_bucket" "default" {
  bucket = var.name
  acl    = "private"

  tags = var.tags
}
```

And his friends, the **Variables.tf** (which just have those two vars up there, nothing magical)... As well as the more important (for this test)...

#### outputs.tf
```
output "bucket_id" {
  value       = aws_s3_bucket.default.id
  description = "Bucket Name"
}

output "bucket_arn" {
  value       = aws_s3_bucket.default.arn
  description = "Bucket ARN"
}
```

In here we are outputting the bucket_id and bucket_arn from the S3 bucket that is going to be spun up.


Alright, boring module is done.

### The test
The test is living inside the *./tests* directory in the repo, and there's a lot more going on here. But similar to the S3 bucket, I've kept it basic so I can iterate on this as I go without confusing myself.

#### s3bucket_test.go
```
func TestS3(t *testing.T) {
	t.Parallel()

	// Give Bucket our Name String
	expectedName := fmt.Sprintf("al-bucket-test-%s", strings.ToLower(random.UniqueId()))
	// Choose a Random Region
	awsRegion := aws.GetRandomStableRegion(t, nil, nil)
```
We start here by opening up the Function as per golang, and configuring some values to supply our build
- `t.Paralell()` is here. <sub>Not going to lie i'm new to golang/terratest expert and my TODO list is finding more about this. All the examples have it so here it is!<sub>
-  We then create a string for our bucket to be named in `expectedName`, in this case I wanted to append some random to the end due to S3 buckets being unique.
- We also chose a random region at `awsRegion`. Terratest has a bunch of builtin functions for randomizing this kind of stuff.

```
	// Begin Creating Options
	terraformOptions := &terraform.Options {
		// The path to where your Terraform code is located
		TerraformDir: "../",

		// Vars to Pass to the Module via -var. Made up from Above Strings
		Vars: map[string]interface{}{
			"name":	expectedName,
		},

		// Environment Variables to run alongside Terraform
		EnvVars: map[string]string{
			"AWS_DEFAULT_REGION": awsRegion,
		},
	  }
```
We start specifying the options to pass into our Terraform commands inside `terraformOptions`
- `TerraformDir` indicates the location of the module.
- `Vars:` is a mapping of various variables to pass on the CLI, in this case we are passing the `expectedName` var into the `name` variable of the Terraform Module
- `envVars` is a mapping of Environment Variables to set on the build. This is where we can consume the random region we asked for with `awsRegion`.

Alright, we have our settings ready, lets configure the job:
```
	  // At the end of the test, run `terraform destroy`
	  defer terraform.Destroy(t, terraformOptions)

	  // Run `terraform init` and `terraform apply`
	  terraform.InitAndApply(t, terraformOptions)

	  // Run `terraform output` to get the value of output variables
	  actualBucketName := terraform.Output(t, terraformOptions, "bucket_id")

```
This one is easy, we basically start running our commands
- `defer terraform.Destroy` allows us to set up our destroy command to stand down the Infrastructure once we are done. It's also instantiating the values from `terraformOptions`.
    - the `defer` here is how golang clean up depending on the run since its evaluated early, but not executed until the completion of the function.
- We then run `terrform.InitAndApply`. Seems obvious, also taking in the `terrformOptions`
- Finally, we grab some specific outputs from the run once the Infrastructure is up. In this case we look up the output `bucket_id` that we specified in our **outputs.tf** file module.


```

	  // Verify we're getting back the outputs we expect
	  assert.Equal(t, expectedName, actualBucketName)
	}
```
And the last step is our assertion. Our true test... In this case it's a very unimpressive test to confirm the Bucket Name we gave it, is the same as the one that was created.


## Running the Test
Now we've set everything up. Let run it.

Pretty basic stuff here, we just change directory into the *./tests* dir, and run:

1. `go mod init *reponame*` - This inits and configures and module requirements
2. `go test` - Runs the test


With that, we get some output.
```
Adams-MacBook-Pro:tests adamlash$ go test
TestS3 2020-03-09T00:39:04-04:00 region.go:91: Using region eu-west-1
TestS3 2020-03-09T00:39:04-04:00 retry.go:72: terraform [init -upgrade=false]
TestS3 2020-03-09T00:39:04-04:00 command.go:87: Running command terraform with args [init -upgrade=false]

...

TestS3 2020-03-09T00:39:05-04:00 command.go:87: Running command terraform with args [get -update]
TestS3 2020-03-09T00:39:05-04:00 retry.go:72: terraform [apply -input=false -auto-approve -var name=al-bucket-test-w9gj6n -lock=false]
TestS3 2020-03-09T00:39:05-04:00 command.go:87: Running command terraform with args [apply -input=false -auto-approve -var name=al-bucket-test-w9gj6n -lock=false]

...

TestS3 2020-03-09T00:39:19-04:00 command.go:158: Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
TestS3 2020-03-09T00:39:19-04:00 command.go:158: 
TestS3 2020-03-09T00:39:19-04:00 command.go:158: Outputs:
TestS3 2020-03-09T00:39:19-04:00 command.go:158: 
TestS3 2020-03-09T00:39:19-04:00 command.go:158: bucket_arn = arn:aws:s3:::al-bucket-test-w9gj6n
TestS3 2020-03-09T00:39:19-04:00 command.go:158: bucket_id = al-bucket-test-w9gj6n
TestS3 2020-03-09T00:39:19-04:00 retry.go:72: terraform [output -no-color bucket_id]
TestS3 2020-03-09T00:39:19-04:00 command.go:87: Running command terraform with args [output -no-color bucket_id]
TestS3 2020-03-09T00:39:19-04:00 command.go:158: al-bucket-test-w9gj6n
TestS3 2020-03-09T00:39:19-04:00 retry.go:72: terraform [destroy -auto-approve -input=false -var name=al-bucket-test-w9gj6n -lock=false]

...

TestS3 2020-03-09T00:39:32-04:00 command.go:158: Destroy complete! Resources: 1 destroyed.
PASS
ok      tf-aws-s3bucket 27.455s
```

There's a lot of chaff I took out but we can see when I ran the test it:
1. Creates our two values for random region and name
2. Does Terraform Init/Apply with the values we provided for Module Locale and Vars/Env Vars
3. Checks the outputs created from Terraform
4. Asserts our test, returns a Pass



## End of the Road
With that, we tested a basic S3 bucket Module. Future Post on this will include iterating on this module while following a true 'TDD' style. Where we write a new test and try to get our Module to comply with these tests.

Overall i'm pretty impressed with Terratest. I did also want to assert it had a mapping of tags, but I could not find an obvious way to do this with S3 buckets (supported with EC2 though..?). So I'm not sure how many shortcomings I'll run into over my time with this, but it's worth giving a crack.