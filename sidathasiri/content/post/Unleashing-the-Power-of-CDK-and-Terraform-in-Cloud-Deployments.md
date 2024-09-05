---
layout: post
title: 'Maximizing Cloud Infrastructure Efficiency with AWS CDK and Terraform'
date: 2024-06-22
author: 'Sidath Munasinghe'
keywords: 'cdk, terraform, cloud, devops, infrastructure as code, sidath, munasinghe'
description: "Uncover the art of integrating CDK and Terraform for seamless synergy in cloud deployments. Explore how these powerful tools streamline infrastructure as code, leading to faster and more efficient cloud deployment processes."
URL: '/2024/06/22/Unleashing-the-Power-of-CDK-and-Terraform-in-Cloud-Deployments/'
image: '/images/posts/Unleashing-the-Power-of-CDK-and-Terraform-in-Cloud-Deployments/main-cover-image.png'
---

Deploying applications to the cloud has become a cornerstone of modern software development. AWS offers CloudFormation as a service to facilitate cloud deployments and tools like the AWS Cloud Development Kit (CDK). At the same time, Terraform has emerged as a powerful solution for Infrastructure as Code (IaC), enabling faster deployments to multiple cloud providers. In this article, we’ll explore the benefits of using AWS CDK and Terraform together and walk through a practical example of creating a REST API with CDK in TypeScript.

## Terraform and CDK

Terraform and CDK are prominent tools that empower the definition of infrastructure as code. Each solution possesses its own set of advantages and disadvantages. Let's delve into a bit more information on both.

### Terraform
Terraform is a tool created by HashiCorp that allows you to define your infrastructure in a high-level configuration language called HCL (HashiCorp Configuration Language). Terraform is cloud-agnostic and can manage infrastructure across various cloud providers, including AWS, Azure, and Google Cloud Platform. It also enables faster deployments when compared to CloudFormation, specifically for AWS.

### CDK
The AWS Cloud Development Kit (CDK) is an open-source software development framework for defining cloud infrastructure in code and provisioning it through AWS CloudFormation. CDK uses familiar programming languages, including TypeScript, to model your applications. Underneath, CDK generates plain CloudFormation templates to create the infrastructure using the code we implement with CDK. The advantage is due to this abstraction, we could generate very lengthy CloudFormation templates within a few lines using high-level CDK constructs. So, it helps developers implement and maintain infrastructure code conveniently with their favourite programming language.

## Advantages of Using Terraform and CDK Together
Using both tools together, we can enjoy the benefits of both worlds. Although Terraform uses HCL, it may not be very convenient for developers. CDK solves this by providing high-level reusable CDK constructs to implement the infrastructure within a few lines. Also, since we use a very familiar programming language, it feels so close to the developers.

On the other hand, CDK uses CloudFormation behind the scenes, which is usually slower than Terraform. However, when we use CDK and Terraform together, we can make much faster cloud deployments since we use Terraform to perform the deployments.

> We can achieve this through the use of [CDK for Terraform](https://developer.hashicorp.com/terraform/cdktf), which is introduced as Cloud Development Kit for Terraform (CDKTF), allowing us to utilise familiar programming languages to define and provision infrastructure.

## Project Setup
Let’s set up a Terraform project with CDK using Typescript as the language. We need to set up a few prerequisites for using CDK for Terraform.

- [Terraform CLI](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- [NodeJS](https://nodejs.org/en)
- [TypeScript](https://www.typescriptlang.org/)
- [CDKTF CLI](https://developer.hashicorp.com/terraform/tutorials/cdktf/cdktf-install)

Once the setup is complete, we can initiate a project. First, let’s create a folder to set up the initial code.

![Creating the project folder](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/30cwddrxv9ix2b0m6iks.png)

Then we can initiate a project with below CLI command. Here we are going to use TypeScript.

![Initializing the project](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8uxk1g1zxa5nxlc0q60i.png)

Once the project is initialized, we can update the main.ts file to define the infrastructure we need. Within the main.ts file, it has created a CDK app as well as a Stack. We can update the resources within the stack as needed to deploy. Let’s build a simple hello world REST API using API Gateway and a Lambda function.

## Creating a REST API
Before adding any AWS resources, let’s configure the AWS provider in Terraform since we will use AWS as the cloud provider. Further, we can use a S3 bucket to store the Terraform backend and track the deployment states.

We can simply configure this by adding the necessary CDK constructs (AwsProvider, S3Backend) with the required parameters like below.

![Configuring AWS provider with a S3 backend](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/doykj03sg8vcytzj1xoq.png)

Here, we have configured the AWS provider by providing the AWS account ID we need to deploy with the region. Similarly, we have configured the S3 backend by providing the bucket name and other configurations.

Now, let’s create an IAM role as the execution role for the Lambda function, including the permissions for a basic lambda execution role.

![Creating an IAM role for the Lambda function](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gdfg5jk1w5od0cd10l0o.png)

Now, it’s time to create the Lambda function. Let’s add the below code for the Lambda function code inside index.ts file within the src folder. Since we are building a simple hello-world application, the Lambda function is returning a simple hello-world response.

![Lambda function handler implementation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wt1ywygtlkcptujhy5cb.png)

Once we have added the Lambda function handler implementation, we can add the CDK implementation to refer to that and create the Lambda function resource.

![Creating the Lambda function resource](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fedzswjgoy0dn5qog55m.png)

The above definition will create a S3 bucket to hold the function code and create the Lambda function. The role we defined earlier is also provided as the execution role for the function.

Once the Lambda function is ready, we can now create the API Gateway REST API and integrate the Lambda function with it.

![Creating the API Gateway REST API](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qxlwb1bz526kuugzyn1x.png)

Here, we are defining the constructs for the API Gateway, a resource for the `/hello` path and the `GET` method under that for the `/hello` `GET` endpoint. Finally, we have integrated it with the lambda function we created earlier as a proxy integration.

Since things are integrated correctly, we can create a stage in the API Gateway and create a deployment like the one below.

![API Gateway stage and deployment](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9pu5d4gvgyh62hbfuca6.png)

We have provided the stage name and API we want to create in the configurations.

Now, we have created all the resources we need. But there is one more thing we need to do. We need to ensure that the API Gateway service can invoke the provided lambda function. To do that, we must create and attach a resource-based policy to allow that action within the Lambda function. We can do it like below easily using the `LambdaPermission` construct.

![Granting permissions to allow lambda invocation from API Gateway](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pnbl4gdfld9a3uyfj9mx.png)

This construct will add the required permissions to the Lambda function to be invoked by the API we created earlier. With this, we are complete with the implementation.

***The full implementation of this project can be found on this [GitHub repository](https://github.com/sidathasiri/cdk-terraform).***

Now everything is ready to deploy. Ensure you have configured your AWS credentials correctly so the Terraform can access AWS to provision the infrastructure. We can first build the code and then deploy it using the command below.

![Building the code and deploying to AWS](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x8e5jn5keqohjirj71w8.png)

Once the above command is executed, CDK for Terraform will install if there are any missing packages and start the deployment. After the deployment is complete, you can verify the created resources and play with the API.

![Testing the deployed API](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fmydxln4kqkll9wkkplz.png)

Moreover, we can discern that Terraform is executing the deployment significantly faster than CloudFormation, which is incredibly advantageous.

To delete the resources you created, you can run the cdktf destroy command. This will ensure that all the resources created by the project are properly cleaned up.

## Conclusion
Using AWS CDK with Terraform offers several notable benefits for managing cloud infrastructure. CDK’s deep integration with AWS and support for familiar programming languages like TypeScript make defining AWS resources intuitive and maintainable. Terraform’s cloud-agnostic capabilities complement CDK by allowing for seamless management across multiple cloud providers. This combination provides flexibility, ease of use, and modularity, enhancing the overall infrastructure management workflow. By leveraging both tools, you can streamline deployments, improve efficiency, and achieve a more robust and versatile infrastructure management solution.
