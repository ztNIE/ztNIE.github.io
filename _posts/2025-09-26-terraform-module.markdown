---
layout: post
title: "The Lessons I Learnt in a Hard Way Building Terraform Modules"
date: 2025-09-26 20:29:43 +1000
categories: blog
---
{% include google_tag.html %}
The principle of building good terraform modules is no different from building any good reusable code/modules in any other languages. There are some lessons that I learnt in the hard way and I'd like to share them here so you don't have to. 

Hope these tips can help you.

## Start with the requirements and common scenarios first.  

There is no way your module is going to cover EVERY scenario out there. If you'd like to do that, you might be better off just using vanilla terraform resources, because, to make it happen, I'm sure there will be some terraform _dark magic_ that takes a price.

Instead, you can:

1. Gather the requirements from the developers.  
Understand the common scenarios they will be using the modules for.
2. Aiming for 95% coverage of the scenarios.  
you will have a much cleaner design, much smaller scope to work on and much __nicer__ first release.
3. Add more functionality to the modules gradually.  
Everyone loves agile, right?


## Use the public modules when possible.  

It's always tempting to re-invent the wheels, that's why everyone knows re-inventing wheels is bad but we're still doing it all the time.

After you get the requirements and understand the scenarios, it's time to look if an existing module can serve you. Maybe all you need is to just figure out a set of vars to use for a public module.

We created our own VPC module. __Don't do that__, just use [one](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) of the thousand VPC modules out there.

Some popular modules:
1. [AWS official modules](https://registry.terraform.io/namespaces/terraform-aws-modules)
2. [Cloud Posse modules](https://docs.cloudposse.com/modules/)

## Design before code.  

It's software engineering 101, but it's so easy to be overlooked, especially when it comes to terraform. After all, it's just a _configuration language_ that people use when they are tired of click-ops, right? 

It might be true. When terraform is used for a small project or even for a local module, it's okay to go rouge a bit, no big deal.

When it comes to a remote terraform module that you'd like to share with __multiple developer teams__ or even __external customers__, it becomes a proper software engineering problem. 

We need to go the whole nine yards -- clean code, separation of concerns, you name it.

Think about:
1. how each module will work with each other, which resource should belong to which module.
2. Defining the vars and outputs of the modules before implementation, as if you're designing APIs. 
   
It would save so much time from changing things back and forth.

## Beware of future breaking changes and one-way doors.

We're aiming for a neat, clean, nice first release that dazzles developers, but be aware of the extensibility of the modules. There are certain decisions that's hard to change later, certain doors are only one-way.

> The definition for "breaking change" of the following discussion is: if a new release needs the user to _change_ their terraform template, like adding `moved, removed` resources, changing variable name, etc., it is a **breaking change**.  
> A minor version release is a version user can safely bump up the number and get a `No changes.` from `terraform plan`.

### Future breaking changes:  

   1. Variable/output name or type change.  
   i.e. From
       ```terraform
       variable "security_group_id" {
           type = string
           description = "The ID of the security group to associate with the resource." 
       }
       ```
       To
       ```terraform
       variable "security_group_ids" {
           type = list(string)
           description = "The IDs of the security groups to associate with the resource."
       }
       ```
   2. Moving resources between modules.
   It requires a `moved` block for users to update.
        ```
        moved {
            from = module.ModuleA.resource.name
            to   = module.ModuleB.resource.name
        }
        ```

### One-way doors:  
1. `lifecycle` rules

You can't use `variable` in `lifecycle` rules, which means you can't make a feature flag to turn on/off the `prevent_destroy` or `ignore_changes`.

Only add `lifecycle` to your module if you're super confident and it's absolutely needed.

Also there are some decisions looking like a big deal, but actually are **safe two-way doors**:

### Two-way door: hard-coded default value in resources. 

It's okay to not make a variable for every attribute in your module. If the developer is not likely to modify it, it's perfectly fine to make it a hard-coded value because it's only a minor release to upgrade that to variable.

For example, say this is the release `v1.0.0`

```terraform
resource "aws_resource" "example" {
    attr_a = var.var_a
    attr_b = "value_b"
}
```

We hard-coded `attr_b` with a sane default `value_b`. Later, we get a feature request to make the `attr_b` a variable. It's as simple as
```terraform
variable "var_b" {
    default = "value_b"
    ...
}

resource "aws_resource" "example" {
    attr_a = var.var_a
    attr_b = var.var_b
}
```

It's a safe minor change. The default value of `var_b` is still `value_b`, so users can safely bump version from `v1.0.0` to `v1.0.1` and will get a `No changes.` after upgrade.

During our implementation, we spent hours on discussing if we could leave something as a hard-coded value, which will be any easy minor release in the future anyway; while greenlighted __one-way door__ without knowing wht it means. 

There is a `prevent_destroy = true` for one of the resource, which has blocked us from having ephemeral environments and even ephemeral integration tests until now.

__Be careful of one-way doors.__


## Variable should have types WHENEVER possible.  
   When I say "WHENEVER" I mean it literally. There are almost no good reasons for not having a type. No excuses, it must have types.   

## Validation.  
[Input validation](https://developer.hashicorp.com/terraform/language/block/variable#validation) is the key to make the modules more usable. If someone is not familiar with your code and is providing some variables won't work, it's better to let them know as early as possible.  
There are several places in terraform that you can validate the input and the outcome.   

### Single Variable.  
Variable validation can be used when you'd like to validate a __single__ variable input. It looks like
```terraform
variable "var_name" {
    ...
    validation {
        condition = "..."
        error_message = "..."
    }
}
```
Please add input validation whenever you can. It was a quite time consuming process, but now, with AI, it's so much easier to create good validations for inputs.

It's especially useful when certain attributes will only fail at `terraform apply` stage. Adding validation could help users catch those errors earlier in the development.

For example, the `bucket_name` of `s3` could not include underscore `_`, and it won't fail until you run `terraform apply`.

Without input validation, for this terraform template
```terraform
# root project
module "Example" {
    source             = "./Example"
    aws_s3_bucket_name = "my_bucket-name"
}

# Example module
variable "aws_s3_bucket_name" {
    description = "The name of the S3 bucket"
    type        = string
}

resource "aws_s3_bucket" "example" {
    bucket = var.aws_s3_bucket_name
}
```
We will get the followings from `validate` and `plan`
```bash
❯ terraform validate
Success! The configuration is valid.

❯ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
+ create
Terraform will perform the following actions:

# module.Example.aws_s3_bucket.example will be created
+ resource "aws_s3_bucket" "example" {
    ...
    + bucket                      = "my_bucket-name"
    ...
}
Plan: 1 to add, 0 to change, 0 to destroy.
```
We will only get errors with `terraform apply`
```bash
module.Example.aws_s3_bucket.example: Creating...
╷
│ Error: validating S3 Bucket (my_bucket-name) name: only lowercase alphanumeric characters and hyphens allowed in "my_bucket-name"
│ 
│   with module.Example.aws_s3_bucket.example,
│   on Example/main.tf line 11, in resource "aws_s3_bucket" "example":
│   11: resource "aws_s3_bucket" "example" {
```
Firstly, it's bad because it's already too late in the pipeline. If the deployment is fully CI/CDed, then you might only get the error after the commit is merged into release branch.

Secondly, there are no `var.aws_s3_bucket_name` mentioned in the error message. For the users, who's not familiar the module implementation, an error message like this means they will have to dig into module's implementation to figure out the root cause.  

With input validation, it will become a much more pleasant experience because the users can catch the error earlier and easier to find where to fix.

```terraform
variable "aws_s3_bucket_name" {
    description = "The name of the S3 bucket"
    type        = string
    validation {
        # Just check if includes "_" as validation example
        condition     = !strcontains(var.aws_s3_bucket_name, "_")
        error_message = "The S3 bucket name must include an underscore '_'."
    }
}
```
Now, if you run `terraform validate` or `plan`, it will give you
```bash
❯ terraform validate
╷
│ Error: Invalid value for variable
│ 
│   on main.tf line 16, in module "Example":
│   16:   aws_s3_bucket_name = "my_bucket-name"
│     ├────────────────
│     │ var.aws_s3_bucket_name is "my_bucket-name"
│ 
│ The S3 bucket name must include an underscore '_'.
│ 
│ This was checked by the validation rule at Example/main.tf:4,3-13.
```
Now the error message contains `var.aws_s3_bucket_name`, which gives a much easier pointer for users to look at. You can also customize the error message to leave more instructions when needed.

### Multiple Variables Validation.  
Chances are sometimes certain attributes only work with each other. For example, a flag `enable_resource` and the configs of that `resource`. You can only refer the the same variable in the input validation expression, so it's not feasible to do something like.  
```terraform
validation {
    condition = var.enable_resource && var.resource_attr_1 == "something"
    ...
}
```

An easy work-around for this is to make multiple vars an object, in this way you can reference multiple values in the same condition expression.
```terraform
variable "s3_setting" {
    type        = object({
        enable_bucket = bool
        bucket_name   = optional(string, null)
    })
    validation {
        condition     = var.s3_setting.enable_bucket == false || (var.s3_setting.enable_bucket == true && var.s3_setting.bucket_name != null)
        error_message = "When 'enable_bucket' is true, 'bucket_name' must be provided."
    }
}

# The variable is used as
resource "aws_s3_bucket" "example" {
    count  = var.s3_setting.enable_bucket ? 1 : 0
    bucket = var.s3_setting.bucket_name
}
```
When you enable s3_bucket but forget to provide the required vars, you'll get
```bash
❯ terraform validate
╷
│ Error: Invalid value for variable
│ 
│   on main.tf line 16, in module "Example":
│   16:   s3_setting = {
│   17:     enable_bucket = true
│   18:   }
│     ├────────────────
│     │ var.s3_setting.bucket_name is null
│     │ var.s3_setting.enable_bucket is true
│ 
│ When 'enable_bucket' is true, 'bucket_name' must be provided.
│ 
│ This was checked by the validation rule at Example/main.tf:8,3-13.
```

You can also use a `check` block if you'd like to keep the vars seperated. However, note that when the `condition` in `check` block fails, it only __WARNs__ you. It won't stop the `plan` or `apply`

```terraform
variable "enable_bucket" {
  type    = bool
  default = false
}

variable "bucket_name" {
  type    = string
  default = null
}

check "bucket_name_provided" {
  assert {
    condition     = var.enable_bucket == false || (var.enable_bucket == true && var.bucket_name != null)
    error_message = "bucket_name must be provided when enable_bucket is true"
  }
}
```
```bash
❯ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # module.Example.aws_s3_bucket.example[0] will be created
  + resource "aws_s3_bucket" "example" {
        ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
╷
│ Warning: Check block assertion failed
│ 
│   on Example/main.tf line 13, in check "bucket_name_provided":
│   13:     condition     = var.enable_bucket == false || (var.enable_bucket == true && var.bucket_name != null)
│     ├────────────────
│     │ var.bucket_name is null
│     │ var.enable_bucket is true
│ 
│ bucket_name must be provided when enable_bucket is true
╵

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.
```


> P.S. There is a reason why sometimes we can't just do
> ```terraform
> resource "aws_s3_bucket" "example" {
>   count  = var.s3_setting.bucket_name != null ? 1 : 0
>   bucket = var.s3_setting.bucket_name
>}
> ```
Terraform needs to know the value of the var used in `count` or `for_each` before the plan. If the var you used has dependency on the values unknown before apply, you might need to do a multi-stage apply for the first time.

### Resource/Output post or pre condition
You can also add [postcondition](https://developer.hashicorp.com/terraform/language/block/resource#postcondition) and [precondition](https://developer.hashicorp.com/terraform/language/block/resource#precondition) to the resources.

I think these are generally more useful for end-users to add in root module level. If you'd like to add post/precondition checks, please be aware of the __one-way doors__ they will introduce.

## Tests.   
[Terraform tests](https://developer.hashicorp.com/terraform/language/tests) is available in Terraform `v1.6.0` and later. We use `command = plan` for unit tests (triggered on every commit) and `command = apply` for integration tests (triggered on every release candidate).

We started without unit tests and had a debate about if we need unit tests when our deadline is close. Investing on testing has proven to be a good investment. Having unit tests for our modules has significantly increased our velocity to develop the modules. Also, it makes it much easier for other developers to contribute to our modules.

## Documentation.  
[Terraform docs](https://github.com/terraform-docs/terraform-docs) is pretty good. To make sure our doc is always up-to-date, we put a `diff` pipeline to trigger on every commit.

The only caveat we had with `terraform docs` is the format -- `markdown table` is quite hard to read when the description of the variable becomes longer, although it looks more similar to most of terraform docs out there.  
`markdown document` renders much better with the default markdown of our code repository.

