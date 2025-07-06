---
layout: post
title: 'Effortless Kubernetes Infrastructure Provisioning and Application Deployment with Cluster.dev and Helm Charts'
# subtitle:   "Hello World, Hello Blog"
date: 2024-02-17
author: 'Sidath Munasinghe'
keywords: 'kubernetes, cluster.dev, aws, amazon, iac, cloud, devops, helm, sidath, munasinghe'
description: 'Learn how to create a Kubernetes cluster on AWS and deploy a Jenkins service via Helm within minutes using Cluster.dev'
URL: '/2024/02/17/Deploy-Kubernetes-in-Minutes-Effortless-Infrastructure-Creation-and-Application-Deployment-with-Cluster.dev-and-Helm-Charts/'
image: '/images/posts/Deploy-Kubernetes-in-Minutes-Effortless-Infrastructure-Creation-and-Application-Deployment-with-Cluster.dev-and-Helm-Charts/k8-with-helm.png'
---

Kubernetes has quickly become the leading orchestration tool for containerized applications, celebrated for its ability to scale applications robustly and resiliently. Its success lies in a powerful framework that adeptly handles the complex life cycles of distributed applications. Yet, deploying Kubernetes cluster infrastructure, essential for tapping into this platform’s capabilities, presents significant challenges. This complexity requires considerable effort and poses a major hurdle for many seeking to leverage Kubernetes for scalable application deployment.

This guide is designed to demystify the process, showing you how to set up a Kubernetes infrastructure swiftly and without hassle. We aim to transform the perceived daunting task of Kubernetes deployment into a streamlined and straightforward process so you don’t want to struggle with its complexities. By the end of this article, you’ll be equipped to utilize the full power of Kubernetes, making infrastructure deployment not just feasible but also simple, efficient, and repeatable. Let’s embark on this journey to simplify Kubernetes deployment, turning obstacles into opportunities for growth and innovation.

## Challenges with Kubernetes
If you have tried to create your own Kubernetes cluster, you know the pain behind the complexities of that. The biggest problem is having a highly available and scalable control plane for cluster management. The easiest way to solve this challenge is to hand over that complexity to a managed service by a cloud provider so that you don’t need to worry about it.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z1m55xa6e3fhkm5ue9vm.png)

However, that’s not the end. Still, you will need to do a lot of configurations to get it working. If you see the [documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) of Elastic Kubernetes Service (EKS), the managed service for Kubernetes from AWS, it has a lot of things to do, like setting up IAM roles, creating certain subnets in the VPC, etc. Although we can use an IaC to provision infrastructure, implementing them would take significant effort. We can resolve this with Cluster.dev using its reusable templates so we don’t need to implement the same code repeatedly. With Cluster.dev, we can get a working Kubernetes cluster effortlessly within a few minutes. If you are not familiar with Cluster.dev, read my previous articles from [here](https://aws.plainenglish.io/revolutionizing-infrastructure-management-with-cluster-dev-a-journey-into-effortless-orchestration-759b9379cebe).

We now have an operational Kubernetes cluster ready to accept deployments. If you are familiar with Kubernetes concepts, to deploy an application, we need to create several Kubernetes resources such as deployments, services, ingress controllers, config maps, stateful sets, etc, as needed, which takes some effort to create the necessary YAML files including configurations. To simplify this, we can use Helm charts.

[Helm](https://helm.sh/) is a package manager that automates Kubernetes applications' creation, packaging, configuration, and deployment by combining your configuration files into a single reusable package. This eliminates the requirement to create the mentioned Kubernetes resources by ourselves since they have been implemented within the Helm chart. All we need to do is configure it as needed to match our requirements. From the public Helm chart repository, we can get the charts for common software packages like Consul, Jenkins SonarQube, etc. We can also create our own Helm charts for our custom applications so that we don’t need to repeat ourselves and simplify deployments.

## Deploying Jenkins with Cluster.dev and Helm on Kubernetes

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ig85dkbwukorl7429d80.png)

Now, let’s see how we can use these tools together to deploy and run a Jenkins service on AWS EKS. Since Cluster.dev natively supports Helm, we can make this even more effortless.

Let’s start by setting up the necessary prerequisites. We need to install the below CLIs to start the deployments. Although there are several tools to install, please note that this is a one-time setup. So we only need to spend time to install them only once.

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli): We use this to provision the AWS infrastructure to run the Kubernetes cluster
- [Cluster.dev](https://docs.cluster.dev/installation-upgrade/): We use this to orchestrate the deployment by using Terraform to deploy the cluster and then use Helm to deploy Jenkins
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html): We use this to interact with our AWS account
- [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html): We use this to interact with the Kubernetes cluster
- [Helm](https://helm.sh/docs/intro/install/): We use helm to simplify the Kubernetes deployments

Once the CLIs are ready, we can generate the Kubernetes infrastructure code and deploy it with Cluster.dev. We can use the below command to reuse a Cluster.dev [template](https://github.com/sidathasiri/cdev-eks) that I have created previously and do the generation instantly.

To do this, create a project folder and run the below Cluster.dev command inside it.

```
cdev project create https://github.com/sidathasiri/cdev-eks.git - interactive
```
This will give you an interactive guide like the one below to configure the template with custom configurations.

![Cluster.dev interactive CLI to generate code
](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8acq7712rc2l96luc1v4.png)

Now, we get a project generated with the necessary configurations to deploy a Kubernetes cluster. Let’s extend it to deploy Jenkins with Helm. To do that, we need to update the `template.yaml` file with additional units as follows. The first unit (kubeconfig) is a shell unit configuring the `kubectl` CLI to interact with the Kubernetes cluster we create. The second unit (jenkins) is a Helm unit configured with the Jenkins Helm chart from the [public Helm repository](https://charts.jenkins.io/). To access the Jenkins service from the public internet, we need to create a Kubernetes service as a load balancer. We can provide this configuration with a `values.yaml` file in `template/helm/values.yaml` path. The second snippet below shows the content of that file.

```
  - name: kubeconfig
      type: shell
      depends_on: this.eksCluster
      force_apply: true
      apply:
        commands:
          - aws eks update-kubeconfig --region {{ .variables.region }} --name {{ .variables.cluster_name }}
  - name: jenkins
    type: helm
    depends_on: this.kubeconfig
    kubeconfig: ~/.kube/config
    source:
      repository: 'https://charts.jenkins.io'
      chart: 'jenkins'
      version: '5.0.13'
    values:
        - file: ./helm/values.yaml
```

```
controller:
  serviceType: LoadBalancer
```

> You can find the full implementation of this from this [GitHub repo](https://github.com/sidathasiri/cdev-eks-jenkins)

That’s all we need to do. Now, we can run the command below, and Cluster.dev will create the Kubernetes cluster and then deploy the Jenkins service within it.

```
cdev apply - force
```
Once the deployment is complete, we can see that a load balancer has been created. We can access the deployed Jenkins service using the DNS address of the load balancer on the port 8080.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gbbd7ybu0hb1s5qps946.png)

See how effortlessly we deployed an application on Kubernetes with minimal code, and even better, in just a few minutes! Say goodbye to the hassle of setting up Kubernetes — we’ve simplified it for you.

## Conclusion
In conclusion, leveraging Cluster.dev alongside Helm charts offers a streamlined approach to Kubernetes infrastructure creation and application deployment. By utilizing Cluster.dev templates, we can efficiently generate infrastructure components, reducing setup time and complexity. Helm charts further simplify the deployment process, allowing for seamless application deployment within minutes. This integration enhances productivity and eliminates tedious manual configuration tasks, making Kubernetes more accessible to developers and teams. Embracing these tools can significantly accelerate the development and deployment lifecycle, enabling focus on core objectives and innovation within Kubernetes environments.

**_Check out my other articles on Cluster.dev series_**

1. [Revolutionizing Infrastructure Management with Cluster.dev: A Journey into Effortless Orchestration](https://medium.com/aws-in-plain-english/revolutionizing-infrastructure-management-with-cluster-dev-a-journey-into-effortless-orchestration-759b9379cebe)
2. [Streamlining SonarQube on AWS ECS: Simplified Deployment Using Cluster.dev](https://aws.plainenglish.io/streamlining-sonarqube-on-aws-ecs-simplified-deployment-using-cluster-dev-0b988536fff0)

## Additional Resources

- [Getting started guide with EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)
- [Cluster.dev documentation](https://docs.cluster.dev/)
- [What is Helm? A complete guide](https://circleci.com/blog/what-is-helm/)
- [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)