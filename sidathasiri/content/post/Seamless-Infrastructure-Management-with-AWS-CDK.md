---
layout: post
title: 'Seamless Infrastructure Management with AWS CDK'
date: 2023-01-20
author: 'Sidath Munasinghe'
keywords: 'aws, ias, cdk, cloud, infrastructure'
description: 'Learn to deploy securely on AWS with GitHub Actions using OpenID Connect. This approach removes the need to manage AWS credentials, bolstering your deployment security.'
URL: '/2023/01/20/Seamless-Infrastructure-Management-with-AWS-CDK/'
image: '/images/posts/Seamless-Infrastructure-Management-with-AWS-CDK/main-logo.png'
relcanonical: 'https://medium.com/99xtechnology/seamless-infrastructure-management-with-aws-cdk-d4fab550c96f'
---

## Introduction

The heart of any software solution is its infrastructure and administering it appropriately is very crucial. At present, it can be seen that the majority of software vendors have moved towards using cloud platforms and serverless architectures to mitigate the complexity of maintaining infrastructure. Still, things like provisioning the infrastructure in diverse environments and keeping them in sync need enormous effort if done manually.

This challenge can be eased by a practice known as Infrastructure as code (IaC) that codifies and manages underlying IT infrastructure. This helps to deploy the infrastructure automatically in a predictable manner by executing a single command wiping out the hurdle of manual work. Further, this facilitates to version control the infrastructure and performing changes in a controlled manner with visibility across the team or organization.

IaC is supported by almost all cloud vendors at present through their own interfaces.

- CloudFormation for AWS
- Resource Manager for Microsoft Azure
- Cloud Deployment Manager for Google Cloud

Often, a declarative definition is used here to outline resources and the configuration via JSON or YAML files. These files can be uploaded to the cloud provider repeatedly varying the necessary configurations to provision different environments quickly and consistently.

## What is AWS CDK

AWS Cloud Development Kit (AWS CDK) is a supplementary interface that permits developers to provision infrastructure on AWS in addition to CloudFormation. In contrast to CloudFormation, CDK allows developers to define infrastructure in a handy manner using popular programming languages like TypeScript. This offer more expressive power to developers to define reusable components and establish the infrastructure of an entire application with a few lines of code.

## Anatomy of a CDK Application

A CDK application can be recognized as a tree of constructs (discussed more below). The bottom of the tree holds low-level constructs symbolizing individual AWS resources while the top of the tree denotes the composition of low-level constructs uncovering the entire infrastructure of the application. This helps to put together a very complex infrastructure favorably using different abstraction layers. At the same time, it helps to strengthen the explainability of the entire application infrastructure.

![Image description](/images/posts/Seamless-Infrastructure-Management-with-AWS-CDK/cdk-anatomy.png)

## Key Concepts

Following are the key concepts that anyone should have when getting started with CDK.

- Construct
- Resource
- Stack
- App

### Construct

The primary element of a CDK application is recognized as a **Construct**. Multiple constructs are combined together to create a stack or an app. [AWS Construct Library](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-construct-library.html) comprises the fundamental constructs representing the AWS resources like S3 bucket, DynamoDB table, AppSync API, etc.

Principally there are 3 levels of constructs,

- **L1 Constructs:** These are very low-level constructs that are accessible from CloudFormation specification. All the configurations that are vital for CloudFormation should be configured here too. Usually, L1 constructs can be easily recognized by their naming convention since all the L1 construct names start with the **Cfn** prefix.

```
new CfnBucket(scope: Construct, id: string, props?: CfnBucketProps)
```

- **L2 Constructs:** These constructs are more high-level constructs than L1 constructs. They offer built-in helper functions and default values for configurations to define the infrastructure with less hassle. The following code snippet illustrates how to create a lambda function and an s3 bucket and grant permissions for the lambda function to access the s3 bucket utilizing the built-in helper functions.

```
    // Level 2 Bucket construct
    const bucket = new s3.Bucket(this, 'uploads-bucket');

    // Level 2 Function construct
    const lambdaFunction = new lambda.Function(this, 'uploads-function', {
      // ...
    });

    // Using helper functions to grant permissions
    bucket.grantPut(lambdaFunction);
    bucket.grantPutAcl(lambdaFunction);
```

- **L3 Constructs:** These are the constructs that have the highest level of abstraction. Often, these constructs are used to implement common patterns in AWS with a few lines of code. The below example demonstrates how to create a REST API with an AWS API gateway backed by a lambda function.

```
// Level 2 Function construct
const lambdaFunction = new lambda.Function(this, 'lambda-function', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'main',
  code: lambda.Code.fromAsset(path.join(__dirname, '../src/handler')),
});

// Level 3 LambdaRestApi construct
const lambdaRestApi = new apigateway.LambdaRestApi(
  this,
  'lambda-rest-api',
  {
    handler: lambdaFunction,
  },
);
```

### Resource

An instance of a construct is entitled to the true AWS resource that will get created by the construct definition. Once the instance (resource) is created, it is feasible to obtain deployed resource attributes of the resource such as the queue URL for an SQS resource to be configured in other dependent resources.

```
const queue = new sqs.Queue(this, 'MyQueue'); // creating the instance/resource
const url = queue.queueUrl; // accessing queueUrl attribute of the created resource

```

### Stack

A set of AWS resources bundled for a single deployment is called a Stack. This is based on the AWS CloudFormation stacks equipping the same features and limitations. When a CDK app is going to deploy, the CDK stack gets synthesized to a CloudFormation template to be deployed to AWS.

### App

App is a special construct that can be only used as the root of a CDK application/construct tree. Due to that, the App construct doesn’t need any initialization argument and it functions as the container for any number of stacks. App construct provides the initial scope to the stacks that it contains and stacks can further pass it down to low-level constructs to create the construct tree.

## Advantages of AWS CDK

There are numerous advantages of using AWS CDK and below are a few of them.

### Customized reusable abstractions

CDK allows defining our own constructs wrapping up the built-in constructs. This authorizes enforcing certain rules to create infrastructure such as tagging standards, naming conventions, etc. Further, the capability to share and reuse these constructs enables to compel those rules across an entire organization reducing code duplication at the same time.

### Power of programming languages

A CDK app can be modeled from multiple languages such as TypeScript, Python, Java, .NET, and Go. This permits using the special capabilities of programming languages such as static type checks, validations, etc going beyond just scripting. This aids to identify issues upfront and fixing them even before an attempt to deploy.

### Confidence

The AWS CDK CLI provides features such as synthesizing a CloudFormation template, showing the differences between the running stack and proposed changes, and confirming security-related changes prior to deployment. Further CDK can be integrated into development tools such as [Visual Studio Code](https://aws.amazon.com/visualstudiocode/) providing more intuition of the application, stacks, resources, policies, etc. All of this help to enhance the confidence in doing changes and evolving the infrastructure of an application.

### Fast development
Development with CDK gets extremely fast mainly due to the ability to reuse constructs. Further, it is more convenient to developers due to the usage of programming languages in contrast to scripts like YAML. Extensions for IDEs/Text Editors such as [CDK Snippets](https://marketplace.visualstudio.com/items?itemName=mklein.cdk-snippets) provide suggestions and auto-completion capabilities to define the infrastructure exceptionally fast.

## Best Practices
- Create custom constructs to impose any standards.
- Develop reusable patterns to avoid repetition.
- Maintain shared constructs/patterns as dependencies to be accessed across the organization.
- Use best practices of programming languages.
- Use AWS CDK tests to enhance resiliency.