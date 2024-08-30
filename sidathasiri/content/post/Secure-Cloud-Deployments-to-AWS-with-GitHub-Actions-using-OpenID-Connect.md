---
layout: post
title: 'Secure Deployments to AWS with GitHub Actions via OpenID Connect'
# subtitle:   "Hello World, Hello Blog"
date: 2023-03-04
author: 'Sidath Munasinghe'
keywords: 'github, github actions, ci/cd, openid connect, aws, cloud'
description: 'Learn to deploy securely on AWS with GitHub Actions using OpenID Connect. This approach removes the need to manage AWS credentials, bolstering your deployment security.'
URL: '/2023/03/04/Secure-Cloud-Deployments-to-AWS-with-GitHub-Actions-using-OpenID-Connect/'
image: '/images/posts/Secure-Cloud-Deployments-to-AWS-with-GitHub-Actions-using-OpenID-Connect/main-logo.png'
---

GitHub Actions is a platform to perform automated tasks upon triggering an event in GitHub. This is being used extensively to govern continuous integration and continuous delivery (CI/CD) workflows once code changes are pushed to a repository. But GitHub Actions has capabilities beyond that to automate a lot of stuff such as tagging releases, sending notifications when creating issues, repository migrations, etc.

## Why GitHub Actions for CI/CD?

There are several benefits to using GitHub Actions for Continuous Integration and Continuous Deployment (CI/CD):

1. **Seamless integration with GitHub:** Since GitHub Actions is a native feature of GitHub, it integrates seamlessly with your code repository. This means you can trigger automated workflows and deployments directly from your repository, without needing to use third-party tools.
2. **Easy automation:** GitHub Actions provides an easy way to automate your CI/CD workflows, allowing you to easily build, test, and deploy your code in a consistent and repeatable way. You can define your workflows using a YAML syntax, and you can trigger them based on various events, such as code commits or pull requests.
3. **Large ecosystem of actions:** GitHub Actions has a large ecosystem of pre-built actions that you can use to build your workflows. These actions can perform tasks like building Docker images, deploying to various cloud platforms, or sending notifications. You can also create your own custom actions if needed.
4. **Cost-effective:** GitHub Actions offers generous free tiers for public and private repositories, making it a cost-effective solution for CI/CD, especially for small and medium-sized projects.
5. **Scalability:** GitHub Actions can scale with your project as it grows. You can use distributed runners to run your workflows on your own infrastructure or on cloud providers like AWS or GCP, allowing you to run multiple workflows in parallel and reduce build times.

_When using GitHub Actions to deploy changes to a cloud provider, the most crucial requirement is providing only the necessary and least access to GitHub to access the cloud provider and execute the deployment successfully. This can be accomplished comfortably by using the OpenID Connect protocol to request access from the cloud provider._

## How Does It Work?

This approach works principally by building a trust relationship between AWS and GitHub and granting GitHub to assume an IAM role with the necessary permissions. This can be fulfilled by defining some rules in AWS to run against the claims included in the OIDC token sent by GitHub. Below are the high-level steps of how this procedure.

1. When the GitHub actions job runs, GitHub’s OIDC Provider generates an OIDC token including a set of claims (ex. repository, repository_owner, job_workflow_ref) and sends it to AWS.
2. AWS verifies whether the received token is valid or not using the pre-exchanged private/public key pairs between AWS and GitHub.
3. If the token is valid, AWS executes the predefined rules against the claims extracted from the token. This is the step where AWS evaluates the trust relationship. The below trust relationship allows any branch, pull request merge branch, or environment from the octo-org/octo-repo organization and repository to assume a role in AWS and no one else.

```
"Condition":
   "StringLike": {
   "token.actions.githubusercontent.com:sub": "repo:octo-org/octo-repo:*"
 }
}
```

4. If the rules are satisfied, AWS issues a short-lived cloud access token that is available only for the duration of the job.

![Image description](/images/posts/Secure-Cloud-Deployments-to-AWS-with-GitHub-Actions-using-OpenID-Connect/overall_diagram.png)

## Integrating to AWS

_This approach works not only with AWS but also with almost any cloud provider like Azure, Google Cloud, and platforms like Hashicorp Vault as well._

Let’s do the configurations that need to be done from AWS end first.

1. Add GitHub OIDC provider to IAM with the correct provider (https://token.actions.githubusercontent.com) and audience (sts.amazonaws.com)
   ![Image description](/images/posts/Secure-Cloud-Deployments-to-AWS-with-GitHub-Actions-using-OpenID-Connect/identity_provider.png)

2. Create an IAM role, as usual, to be assumed by GitHub with the appropriate permissions. In order to make sure that this role can be assumed only by allowed entities in GitHub, the trust relationship rules should be defined under the trust relationship section of the IAM role. All available claims that can be used to define these rules can be found [here](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token).
   ![Image description](/images/posts/Secure-Cloud-Deployments-to-AWS-with-GitHub-Actions-using-OpenID-Connect/github_role.png)

With the above changes, AWS is now ready to allow the created role when a valid token is received from GitHub that fulfills the defined trust relationship.

Now let’s do the changes from the GitHub end. This [configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action has been created by AWS to simplify this process to exchange the OIDC token for a cloud access token. The following code snippet shows a simple example workflow that copies a file to a S3 bucket upon a push event to the git repository. Notice how configure-aws-credentials action is configured with role name, role session name and the was region.

```
name: AWS example workflow
on:
 push
env:
 BUCKET_NAME : “example-bucket-name”
 AWS_REGION : “us-east-1”
# permission can be added at job level or workflow level
permissions:
 id-token: write # This is required for requesting the JWT
 contents: read # This is required for actions/checkout
jobs:
 S3PackageUpload:
 runs-on: ubuntu-latest
 steps:
 — name: Git clone the repository
   uses: actions/checkout@v3
 — name: configure aws credentials
   uses: aws-actions/configure-aws-credentials@v1
   with:
     role-to-assume: arn:aws:iam::1234567890:role/example-role
     role-session-name: samplerolesession
     aws-region: ${{ env.AWS_REGION }}
 # Upload a file to AWS s3
 — name: Copy index.html to s3
   run: aws s3 cp ./index.html s3://${{ env.BUCKET_NAME }}/
```

Now whenever the workflow is triggered, configure-aws-credentials action will send the generated OIDC token to AWS to get the short-lived access token that grants the permission included in the requested IAM role.

## Advantages of Using OpenID Connect

1. No need to create AWS users and expose their AWS security credentials
2. No need to manually rotate credentials to enhance security
3. Provide granular-level access for authentication and authorization management
4. Centralized authentication and authorization

## Conclusion

In this article, we discussed how to use GitHub Actions to securely deploy your applications to AWS using OpenID Connect. You can find the below links to find more details.

## Learn More

- [Quick Start Guide for GitHub Actions](https://docs.github.com/en/actions/quickstart)
- [GitHub Actions Market Place](https://github.com/marketplace?type=actions)
- [GitHub Actions Community Forum](https://github.com/orgs/community/discussions/)