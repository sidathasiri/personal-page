---
layout: post
title: 'CDK for Terraform (CDKTF) in Practice: Building a Serverless REST API with TypeScript'
date: 2024-06-22
author: 'Sidath Munasinghe'
keywords: 'cdktf, cdk for terraform, aws cdk, terraform, typescript, api gateway, lambda, infrastructure as code, devops, sidath, munasinghe'
description: "Combine AWS CDK-style TypeScript with Terraform's fast deployments using CDKTF. Build and deploy an API Gateway and Lambda REST API step by step, with gotchas."
URL: '/2024/06/22/Unleashing-the-Power-of-CDK-and-Terraform-in-Cloud-Deployments/'
image: '/images/posts/Unleashing-the-Power-of-CDK-and-Terraform-in-Cloud-Deployments/main-cover-image.png'
---

What if you could write your infrastructure in TypeScript, the way you would with the AWS CDK, but have Terraform deploy it instead of CloudFormation? That's exactly what CDK for Terraform offers, and in this post I'll show you why the combination is worth a look and then build a small serverless REST API with it from start to finish.

## Two approaches to infrastructure as code

There are plenty of IaC tools, but two come up all the time in AWS projects. On AWS, CloudFormation is the native deployment engine, and the AWS CDK sits on top of it. Terraform is the popular alternative that works across clouds. Each has clear strengths and weaknesses.

**Terraform**, from HashiCorp, describes infrastructure in its own declarative language, HCL (HashiCorp Configuration Language). It isn't tied to one cloud: the same tool and workflow can manage AWS, Azure, Google Cloud Platform and many others through providers. On AWS specifically, Terraform deployments also tend to finish faster than the equivalent CloudFormation stack updates.

**AWS CDK** (Cloud Development Kit) is an open-source framework that lets you describe cloud resources in general-purpose languages such as TypeScript. It doesn't deploy anything itself. Instead, it *synthesises* your code into ordinary CloudFormation templates, and CloudFormation creates the resources. The benefit of this extra layer is that a high-level construct can expand a few lines of code into a very long template, so developers can write and maintain infrastructure in a language they already know.

Here's how they compare side by side:

| | Terraform (HCL) | AWS CDK |
|---|---|---|
| Authoring language | HCL, a domain-specific language | TypeScript, Python, Java, C#, Go |
| Deployment engine | Terraform | CloudFormation |
| Cloud coverage | Multi-cloud via providers | AWS-focused |
| Abstraction | Modules | Reusable, high-level constructs |
| Typical deployment speed on AWS | Faster | Slower |

## The case for combining them

The weaknesses of each tool are mostly the other tool's strengths:

- HCL works, but many developers find it less comfortable than a programming language they use every day. CDK gives you that language, plus reusable constructs that keep infrastructure code short.
- CDK's reliance on CloudFormation makes deployments slower. If Terraform does the deployment instead, you keep the developer experience and get Terraform's speed.

The bridge between them is [CDK for Terraform](https://developer.hashicorp.com/terraform/cdktf) (CDKTF). You write constructs in a familiar language, CDKTF synthesises them into Terraform configuration, and Terraform provisions the resources.

## Building a hello-world API with CDKTF

Our target is a minimal REST API: one `GET /hello` endpoint on API Gateway, backed by a Lambda function that returns a hello-world response.

### Prerequisites

Before you start, install:

- [Terraform CLI](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- [NodeJS](https://nodejs.org/en)
- [TypeScript](https://www.typescriptlang.org/)
- [CDKTF CLI](https://developer.hashicorp.com/terraform/tutorials/cdktf/cdktf-install)

### Scaffolding the project

Start with an empty directory for the code:

![Terminal commands that create and enter a new project directory](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/30cwddrxv9ix2b0m6iks.png)

Then initialise a CDKTF project in it, choosing TypeScript as the language:

![Running the CDKTF init command with the TypeScript template](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8uxk1g1zxa5nxlc0q60i.png)

The generated `main.ts` already contains a CDKTF app and a single stack. Everything we build goes inside that stack.

### Provider and remote state

Since AWS is our target, the first thing to set up is the Terraform AWS provider. I also want Terraform's state stored remotely in an S3 bucket rather than on my laptop, so deployments can be tracked reliably. Both are just constructs (`AwsProvider` and `S3Backend`) in the stack:

![TypeScript code configuring the AwsProvider with account and region, and an S3Backend for Terraform state](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/doykj03sg8vcytzj1xoq.png)

The provider gets the target region and the AWS account ID to deploy into. The backend gets the bucket name and the related state settings.

### Lambda execution role

The function needs an IAM role it can assume, with the permissions of the basic Lambda execution role attached:

![TypeScript code defining an IAM role for Lambda with the basic execution policy attached](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gdfg5jk1w5od0cd10l0o.png)

### The function itself

The handler lives in `src/index.ts`. Since this is a hello-world app, all it does is return a fixed greeting:

![Lambda handler in src/index.ts returning a hello-world response](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wt1ywygtlkcptujhy5cb.png)

Next, we tell the stack about that code and create the Lambda resource:

![CDKTF code that packages the handler, uploads it to S3 and creates the Lambda function with the IAM role](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fedzswjgoy0dn5qog55m.png)

This creates an S3 bucket for the function's code package, uploads the code and creates the function, using the role from the previous step as its execution role.

### Wiring up API Gateway

With the function in place, we can expose it through an API Gateway REST API:

![CDKTF code creating the REST API, the /hello resource, the GET method and a Lambda proxy integration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qxlwb1bz526kuugzyn1x.png)

There are four pieces here: the REST API itself, a `/hello` resource, a `GET` method on that resource, and a proxy integration that sends requests to our Lambda function.

An API isn't reachable until it has been deployed to a stage, so we add a deployment and a stage, giving the stage a name and linking it to the API:

![CDKTF code creating an API Gateway deployment and a named stage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9pu5d4gvgyh62hbfuca6.png)

### Allowing API Gateway to call Lambda

It's easy to forget this last step. Even with the integration defined, API Gateway can't invoke the function until the function's resource-based policy allows it. The `LambdaPermission` construct adds that permission for our API:

![LambdaPermission construct that lets the API Gateway API invoke the Lambda function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pnbl4gdfld9a3uyfj9mx.png)

That completes the infrastructure code.

***You can browse the complete project in this [GitHub repository](https://github.com/sidathasiri/cdk-terraform).***

### Deploying and testing

Check that your AWS credentials are configured so Terraform can reach your account. Then build the project and deploy it:

![Terminal commands that build the TypeScript project and run the CDKTF deploy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x8e5jn5keqohjirj71w8.png)

CDKTF installs any missing packages, synthesises the Terraform configuration and runs the deployment. When it finishes, call the endpoint to confirm everything works:

![Calling the deployed /hello endpoint and receiving the hello-world response](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fmydxln4kqkll9wkkplz.png)

One thing you'll notice straight away is how fast the deployment is compared with pushing a similar stack through CloudFormation. That speed makes a real difference when you iterate many times a day.

When you're done, `cdktf destroy` removes every resource the project created.

## Gotchas and practical tips

- **The state bucket must exist first.** `S3Backend` stores state in a bucket you name, but it doesn't create that bucket. Create it (ideally with versioning enabled) before your first deploy.
- **Don't hard-code the account ID.** Keeping the account ID in the provider configuration is a useful safety check, but read it from configuration or environment variables if you deploy to more than one account.
- **Changing the API needs a new deployment.** API Gateway serves whatever was last *deployed* to a stage. If you add routes later, make sure a new deployment is triggered, or the stage will keep serving the old version.
- **Watch the `LambdaPermission` source ARN.** If the source ARN doesn't match your API, you'll get a "permission denied" style error from API Gateway even though the integration looks correct.
- **Constructs aren't interchangeable.** CDKTF uses its own provider bindings (for example, `@cdktf/provider-aws`). Constructs from the AWS CDK's `aws-cdk-lib` can't be used directly, so expect to write some things at a lower level than you would in the AWS CDK.

## Is CDKTF right for you?

**Good fit:** teams that already use Terraform but want to write infrastructure in a real programming language, teams that like the CDK approach but want faster deployments or resources outside AWS, and projects that share TypeScript between application and infrastructure code.

**Think twice:** if your team is happy with plain HCL and modules, CDKTF adds a synthesis step and another toolchain to maintain. If you rely heavily on the AWS CDK's high-level L2/L3 constructs, you'll miss them. And before committing to it long term, check the current support status of CDKTF on HashiCorp's site.

## Key takeaways

Combining CDK-style development with Terraform gives you a strong workflow for cloud infrastructure. You define resources in a language like TypeScript, which keeps the code readable and maintainable, and you get Terraform's cloud-agnostic reach and faster deployments. The result is flexible, modular and convenient for developers, and it makes provisioning infrastructure faster and more dependable day to day.
