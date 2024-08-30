---
layout: post
title: 'Feature Flags on AWS Using AppConfig for Seamless Software Evolution'
# subtitle:   "Hello World, Hello Blog"
date: 2023-12-20
author: 'Sidath Munasinghe'
keywords: 'feature flags, aws, appconfig, cdk, release, continuos deployment'
description: 'Learn what are feature flags and how to use AWS AppConfig service to implement feature flags. Get hands on experience by integrating feature flags with AWS Lambda using CDK'
URL: '/2023/12/20/Harnessing-Feature-Flags-on-AWS-AppConfig-for-Seamless-Software-Evolution/'
image: '/images/posts/Harnessing-Feature-Flags-on-AWS-AppConfig-for-Seamless-Software-Evolution/main-logo.png'
---

In the fast-paced realm of software development, agility and adaptability are not just a virtue but a necessity. Feature flagging is a versatile technique that offers a strategic approach for releasing features to end users, enabling rapid software development while giving more control to teams to evolve products.

Feature flags decouple feature deployment from code deployment, allowing teams to release features incrementally and independently. Due to this, teams can do frequent code deployments darkly, although the entire feature development is incomplete. Once the entire feature development is complete and ready to enable users, it can be released on demand as needed. This promotes a continuous deployment approach, enabling faster time-to-market without disrupting the entire application.

## Feature Flags

Incorporating feature flags introduces a lot of advantages to the software development process as well as to the software releases.

- **Risk Mitigation:** Feature flags can act as safety nets since they enable teams to turn off features in production if there are any issues without needing additional code deployments. This helps teams to troubleshoot issues and apply the fixes while keeping the production environment healthy.
- **Personalization and Experimentation:** Feature flags provide the capability to enable features for specific users or user segments. This provides the canary feature enablement to roll out new features gradually. Further, this extends to conducting A/B testing and experimentation, where teams can gather valuable user feedback and data to make informed decisions about feature improvements.
- **Unblock Dependencies:** It’s common in software development that teams get blocked due to dependencies with other teams. With feature flags, teams can work independently based on the design/contract and deploy even the partially completed implementation to production behind a feature flag. Once all teams have completed the implementation and everything is tested, it can be enabled for end users.
- **Graceful Degradation:** There are situations where systems receive excessive traffic, putting them under stress and leading to performance issues. In those scenarios, teams can use feature flags to turn off non-critical features and let the core features perform as expected.

## AppConfig Feature Flags

![Source: https://gallery.ecr.aws/aws-appconfig/aws-appconfig-agent](/images/posts/Harnessing-Feature-Flags-on-AWS-AppConfig-for-Seamless-Software-Evolution/app-config.png)

AWS AppConfig is a service that facilitates deploying and managing application configurations on the AWS cloud, including feature flags. It allows teams to create, manage, and deploy configurations seamlessly with several additional capabilities.

- **Rollout strategies:** AWS AppConfig supports different rollout strategies to enable feature activation in a controlled manner. This helps to identify potential issues, gather user feedback in a controlled manner, and react accordingly to keep rolling out or revert.

![Different rollout options in AppConfig feature flags](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kz5efaiiu9y2x4gfuqvb.png)

- **Versioning:** The service supports automatic versioning of configurations, allowing teams to track the entire history of the configuration changes.
- **Rollback:** When there is a need to change the flag status, AppConfig provides a built-in capability to rollback configuration to previous versions.
- **Manage Environments:** AppConfig has a concept called environment, which can be used to control feature flag values per each product environment. This helps to test the feature flag capabilities in non-prod environments first and then use them in production as needed.

## Retrieving State in AppConfig Feature Flags

The real magic behind the feature flags is we don’t need to redeploy applications to get the latest configuration changes. When we update the configuration settings, all the applications will get the latest changes automatically. Applications can get this capability by establishing a configuration session with the AppConfig server and polling for configuration changes. The sequence diagram below summarizes the overall communication flow.

![Sequence diagram for fetching feature flag configurations](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/976wsk6liuyygurgpnfl.png)

1. Create a configuration session using the application name, configuration profile, and environment we must connect to. In response, it will return a token to get the latest configuration in the next poll request.
2. Once the application gets that token, it can be used to get the latest configuration by calling the GetLatestConfiguration API. In response, it will return the latest configuration settings and a new token for the next poll.
3. The application can periodically call the GetLatestConfiguration API using the token that was retrieved previously.

Here, the important thing is that the token can be used only once. So, the application needs to keep track of the latest token it received and use it in the subsequent request.

## Available Integration Options

Since implementing this polling mechanism to get the flag status/configuration is not as straightforward, AWS has provided simplified integration options for common compute services.

### AWS Lambda

AWS Lambda can be easily integrated with AppConfig feature flags with **AWS AppConfig Agent Lambda extension** as a layer to the lambda function. The lambda extension layer handles the polling implementation, and developers can benefit from feature flags. Further, the lambda layer will fetch and store the flag's statuses in a local cache, including the necessary tokens for subsequence API calls.

To access its configuration data, the function can call the AWS AppConfig extension at an HTTP endpoint running on `localhost:2772`. The following diagram shows how it works.

![Source: https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-lambda-extensions.html](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g0osh3m00nccfb0b20li.png)

### Amazon EC2

Like the lambda extension, an **AWS AppConfig Agent** can be installed on the instance to interact with AppConfig and fetch and cache the flag configuration data on your behalf. As we saw earlier, the retrieved configurations can be accessed from `localhost:2772`.

It is important to note that the agent is available for Linux operating systems running kernel version 4.15 or greater, and the agent can be installed via the **yum** command-line package-management utility.

### Amazon ECS and Amazon EKS

AWS AppConfig can be integrated with AWS ECS and EKS using **AWS AppConfig Agent**. The agent functions as a sidecar container running alongside the main container application. The agent will manage all the interactions with AppConfig, and the fetched configurations can be accessed from `localhost:2772`.

## Integrating into AWS Lambda

Now, let’s walk through how to implement this for AWS Lambda, a common serverless computing service. We will use AWS CDK for the infrastructure code to automate the infrastructure provisioning.

To implement this setup, we need to create the below resources on AWS.

- AppConfig Application
- AppConfig Environment
- AppConfig Configuration Profile
- AppConfig Configuration Version
- Lambda Function

Let’s see each component and how to implement them with CDK constructs.

### Creating AppConfig Application

An AppConfig Application refers to a logical entity that utilizes AWS AppConfig for managing its configuration settings. This could be any software application or service that benefits from dynamic and centralized configuration management.

We can create an application with CDK using the code snippet below by providing a meaningful name for the use case.

```
const application = new CfnApplication(scope, `AppConfig Application`,
  name: 'e-commerce-app',
});
```

### Creating AppConfig Environment

An AppConfig Environment is a deployment environment within an AWS AppConfig application where the configurations are managed independently from each other. Environments provide a way to separate configurations for different stages of development, testing, and production, allowing for controlled and efficient management of configurations across different deployment scenarios.

We can create an environment using the snippet below. Since we are creating an environment under an application, we need to provide a reference to the application we created earlier in addition to the environment name.

```
const environment = new CfnEnvironment(scope, `AppConfig Environment`, {
  applicationId: application.ref,
  name: 'DEV',
});
```

### Creating AppConfig Configuration Profile

A configuration profile helps to define what kind of configuration (feature flag/freeform) will be created and optionally defines any validators to ensure the configuration data is syntactically and semantically correct.

The code snippet below creates a **feature flag type** configuration profile called login. When we use the feature flag type, the location URI has to be set with **hosted**. Further, we need to specify the application for this configuration profile.

```
const configurationProfile = new CfnConfigurationProfile(
  scope,
  `AppConfig ConfigurationProfile`,
  {
    applicationId: application.ref,
    locationUri: 'hosted',
    name: 'login',
    type: 'AWS.AppConfig.FeatureFlags',
   }
);
```

### Creating AppConfig Configuration Version

An AppConfig Configuration Version represents a specific snapshot or version of a configuration. As configurations may evolve over time, different versions allow for tracking and managing changes. Each version is associated with a unique identifier.

Using the code snippet below, we create a configuration version for a feature flag called **sso_enabled**, which the value has set to **true**. Similar to the previous constructs, we need to connect this with the application and the configuration profile we need to use.

```
const configurationVersion = new CfnHostedConfigurationVersion(
  scope,
  `AppConfig ConfigurationProfileVersion`,
  {
    applicationId: application.ref,
    configurationProfileId: configurationProfile.ref,
    contentType: 'application/json',
    content: JSON.stringify({
      flags: {
        flagkey: {
          name: 'sso_enabled',
        },
      },
      values: {
        flagkey: {
          enabled: true,
        },
      },
       version: '1',
    })
  }
);
```

### Creating Lambda Function

Now, we can create the lambda function and attach the lambda layer to integrate it with AppConfig. The lambda function must be configured with proper environment variables so the agent can connect with the required AppConfig application, environment and configuration profile, as shown in the code snippet below. Further, we must grant permissions via IAM to the lambda function to access AppConfig and fetch the configurations.

Below is the CDK infrastructure code for the lambda function, including the lambda layer integration and permission granting to AppConfig.

```
const lambdaFunction = new NodejsFunction(this, 'my-lambda-fn', {
      entry: join(__dirname, '../src/lambdaHandler.ts'),
      runtime: Runtime.NODEJS_18_X,
      handler: 'handler',
      timeout: Duration.seconds(5),
      environment: {
        APPCONFIG_APPLICATION_ID: application.ref,
        APPCONFIG_ENVIRONMENT: environment.name,
        APPCONFIG_CONFIGURATION_ID: configurationProfile.ref
      },
});

lambdaFunction.addLayers(
  LayerVersion.fromLayerVersionArn(
    this,
    'AppConfigExtension',
    'arn:aws:lambda:us-east-1:027255383542:layer:AWS-AppConfig-Extension:128'
  )
);

lambdaFunction.role?.attachInlinePolicy(
  new Policy(this, 'PermissionsForAppConfig', {
    statements: [
      new PolicyStatement({
        actions: [
          'appconfig:StartConfigurationSession',
          'appconfig:GetLatestConfiguration',
        ],
        resources: ['*'],
      }),
    ],
  })
);
```

Now, we should be able to access `localhost:2772` from the lambda handler function and get the configuration of the feature flag. The code snippet below shows a very basic implementation. In the request URL, we need to specify the AppConfig configurations we exposed as the environment variables to fetch the configuration from the correct application, configuration profile and the environment we want.

```
import { Handler } from 'aws-cdk-lib/aws-lambda';

export const handler: Handler = async (event: any) => {
  const applicationId = process.env.APPCONFIG_APPLICATION_ID;
  const environment = process.env.APPCONFIG_ENVIRONMENT;
  const configurationId = process.env.APPCONFIG_CONFIGURATION_ID;

  const url = `http://localhost:2772/applications/${applicationId}/environments/${environment}/configurations/${configurationId}`;

  try {
    const response = await fetch(url);
    const responseData = await response.json();

    console.log('data:', JSON.stringify(responseData));
  } catch (error) {
    console.log(console.error);
  }
};
```

Finally, when we test the lambda function from the AWS console, we can see that the AppConfig agent has started and the configuration values have fetched successfully from the CloudWatch logs.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kbholjgwq4gy96rzlat3.png)

> The full implementation of this can be found on this [GitHub repo](https://github.com/sidathasiri/appconfig-feature-flags).

## Conclusion

In conclusion, adopting feature flags, particularly through AWS AppConfig, empowers software teams to achieve a smooth and adaptive software evolution. The advantages of feature flags, such as controlled rollout, risk mitigation, and dynamic configuration, are effectively harnessed using AppConfig, offering real-time control and seamless integration with AWS services.

Different integration options underscore AppConfig’s flexibility in catering to diverse development environments. Altogether, leveraging feature flags on AWS AppConfig enhances development agility, allowing teams to respond dynamically to evolving requirements and user feedback, ultimately fostering a resilient and user-centric software evolution.

## Learn More

- [Retrieving configuration data using the AWS AppConfig Agent Lambda extension](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-lambda-extensions.html)
- [Retrieving configuration data from Amazon EC2 instances](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-ec2.html)
- [Retrieving configuration data from Amazon ECS and Amazon EKS](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-containers-agent.html)
- [Sample implementation with Node SDK](https://github.com/sidathasiri/appconfig-feature-flags)
