---
layout: post
title: 'From Zero to Jenkins on Amazon EKS in Minutes: A Cluster.dev and Helm Walkthrough'
# subtitle:   "Hello World, Hello Blog"
date: 2024-02-17
author: 'Sidath Munasinghe'
keywords: 'kubernetes, amazon eks, cluster.dev, helm charts, jenkins, aws, infrastructure as code, devops, sidath, munasinghe'
description: 'Spin up an Amazon EKS cluster and install Jenkins with a Helm chart in minutes. A hands-on Cluster.dev template walkthrough, plus pitfalls and cleanup tips.'
URL: '/2024/02/17/Deploy-Kubernetes-in-Minutes-Effortless-Infrastructure-Creation-and-Application-Deployment-with-Cluster.dev-and-Helm-Charts/'
image: '/images/posts/Deploy-Kubernetes-in-Minutes-Effortless-Infrastructure-Creation-and-Application-Deployment-with-Cluster.dev-and-Helm-Charts/k8-with-helm.png'
---

Most people who love Kubernetes love it for what happens *after* the cluster exists: self-healing workloads, horizontal scaling, and a consistent way to manage the life cycle of distributed applications. The part nobody enjoys is getting a production-grade cluster up in the first place. That initial setup is where a lot of teams lose days, and sometimes their enthusiasm for Kubernetes altogether.

In this post, I want to show that it doesn't have to be that way. We'll go from an empty AWS account to a running Kubernetes cluster with Jenkins installed on it, using two tools that each remove a big chunk of the work: **Cluster.dev** for the infrastructure and **Helm** for the application. The goal is a setup that is quick the first time and just as easy to repeat the next time.

## Why standing up Kubernetes is harder than it looks

Anyone who has built a cluster by hand knows the hardest piece is the control plane. Keeping it highly available and able to scale is serious operational work. The sensible move is to let a cloud provider run it for you as a managed service, so the control plane becomes their problem rather than yours.

![Diagram contrasting a self-managed Kubernetes control plane with one run by a cloud provider as a managed service](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z1m55xa6e3fhkm5ue9vm.png)

A managed control plane solves only part of the problem, though. Look at the [Amazon EKS getting started documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) and you'll find a long checklist: IAM roles for the cluster and the nodes, a VPC with the right subnets, node groups, and more. You could write all of that in an infrastructure-as-code tool yourself, but it's a lot of code to build, and you'd end up writing much the same code for every new cluster.

This is where Cluster.dev helps. It lets you package infrastructure as reusable templates, so a working EKS cluster becomes something you generate and configure rather than write. (If Cluster.dev is new to you, I introduced it in an [earlier article](https://aws.plainenglish.io/revolutionizing-infrastructure-management-with-cluster-dev-a-journey-into-effortless-orchestration-759b9379cebe).)

## Where Helm fits in

Having a cluster is step one. To actually run an application on it, you normally write a set of Kubernetes manifests (Deployments, Services, Ingress, ConfigMaps, StatefulSets and so on), each with its own YAML and configuration. For a non-trivial application, that adds up fast.

[Helm](https://helm.sh/) is the package manager for Kubernetes, and it takes most of that work away. A Helm chart bundles all the manifests an application needs into one versioned, configurable package. Instead of writing those resources yourself, you install the chart and override only the values you care about. Charts for popular tools such as Jenkins, Consul and SonarQube are published in public repositories, and you can write charts for your own services too, so every deployment follows the same pattern.

Put the two together and the split is simple:

| Layer | Tool | What it handles |
|---|---|---|
| Infrastructure | Cluster.dev (driving Terraform) | VPC, IAM, EKS cluster and nodes |
| Cluster access | Shell unit in Cluster.dev | Configuring `kubectl` for the new cluster |
| Application | Helm (via Cluster.dev's Helm unit) | Installing and configuring Jenkins |

Because Cluster.dev supports Helm natively, a single command can run all three layers.

## Hands-on: Jenkins on EKS

![Architecture of the setup: Cluster.dev provisions an EKS cluster on AWS and installs Jenkins on it with a Helm chart](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ig85dkbwukorl7429d80.png)

### Step 1: Install the tooling (once)

It looks like a lot of tools, but you only install them once and then reuse them for every project:

- [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) provisions the AWS resources underneath the cluster.
- [Cluster.dev](https://docs.cluster.dev/installation-upgrade/) orchestrates the whole flow, calling Terraform for the cluster and Helm for Jenkins.
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) talks to your AWS account.
- [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) talks to the Kubernetes cluster.
- [Helm](https://helm.sh/docs/intro/install/) handles the Kubernetes packaging.

### Step 2: Generate the cluster project from a template

Instead of writing EKS infrastructure code, I'll reuse a Cluster.dev [template](https://github.com/sidathasiri/cdev-eks) I built earlier. Make an empty project folder, move into it, and run:

```
cdev project create https://github.com/sidathasiri/cdev-eks.git --interactive
```

Cluster.dev then asks you a few questions, such as the region and cluster name, and fills the template in with your answers.

![Cluster.dev interactive prompts asking for the values used to generate the EKS project](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8acq7712rc2l96luc1v4.png)

When it finishes, you have a project that can already deploy a Kubernetes cluster.

### Step 3: Add Jenkins to the stack

Now we extend the generated `template.yaml` with two more units:

1. **`kubeconfig`**: a shell unit that runs after the EKS cluster is created and points your local `kubectl` at it.
2. **`jenkins`**: a Helm unit that waits for `kubeconfig` and then installs the Jenkins chart from the [official Jenkins chart repository](https://charts.jenkins.io/), pinned to a specific version.

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

By default, the chart doesn't expose Jenkins to the internet. To reach it from a browser, we ask for its Kubernetes Service to be of type `LoadBalancer`, which makes AWS provision a load balancer in front of it. That override goes in `template/helm/values.yaml`:

```
controller:
  serviceType: LoadBalancer
```

> The complete working project is available in this [GitHub repo](https://github.com/sidathasiri/cdev-eks-jenkins).

### Step 4: Deploy everything with one command

That's the whole configuration. Kick off the deployment:

```
cdev apply --force
```

Cluster.dev works through the units in dependency order: it creates the EKS cluster first, then configures `kubeconfig`, then installs Jenkins with Helm. When it finishes, you'll see a new load balancer in your AWS account. Open its DNS name on port 8080 and the Jenkins UI should load.

![Jenkins web interface loaded through the DNS name of the AWS load balancer on port 8080](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gbbd7ybu0hb1s5qps946.png)

That's a real application running on a managed Kubernetes cluster, from a small amount of configuration and in a few minutes.

## Gotchas worth knowing

A few things that are easy to trip over with this setup:

- **Cost keeps running.** An EKS control plane, worker nodes, NAT gateways (if your VPC uses them) and a load balancer all bill by the hour. When you're done experimenting, run `cdev destroy` from the project folder so Cluster.dev tears down everything it created.
- **A public Jenkins is exposed to the whole internet.** `serviceType: LoadBalancer` is fine for a demo. For anything real, put Jenkins behind an Ingress with TLS, limit source IPs, or keep it internal.
- **Get the initial admin password.** The Jenkins chart generates an admin password and stores it in a Kubernetes Secret, so look it up with `kubectl` before you try to log in.
- **The load balancer takes a moment.** The DNS name can take a few minutes to start resolving after the deployment finishes, so don't assume something broke if the page doesn't load straight away.
- **Pin chart versions.** The `version: '5.0.13'` line keeps deployments reproducible. If you leave it out, you'll get whatever version is newest on the day you deploy.
- **Check your AWS identity first.** `aws eks update-kubeconfig` and the Terraform steps use whatever credentials and region your AWS CLI is set to, so confirm them before you apply.

## When this approach is a good fit (and when it isn't)

It works well when you need to stand up clusters repeatedly (per environment, per team or for short-lived testing), when you want infrastructure and application to be deployed together, and when off-the-shelf charts cover most of your software.

It's less suitable if your organisation already has a mature platform pipeline (for example, Terraform modules plus a GitOps tool such as Argo CD or Flux) or if you need very fine-grained control over each EKS setting. In that case, the template becomes something you have to keep in sync with your existing standards.

## Wrapping up

Pairing Cluster.dev with Helm cuts the cost of running something on Kubernetes at both ends. Cluster.dev templates turn the EKS groundwork into a short generation step, and Helm charts replace hand-written manifests with a configurable package. Because the whole stack is deployed by one tool in the right order, there's much less manual configuration, setups are easy to reproduce, and Kubernetes becomes accessible to developers who don't want to become cluster specialists. You spend less time on plumbing and more on the applications that actually run on the cluster.

**_More from my Cluster.dev series_**

1. [Revolutionizing Infrastructure Management with Cluster.dev: A Journey into Effortless Orchestration](https://medium.com/aws-in-plain-english/revolutionizing-infrastructure-management-with-cluster-dev-a-journey-into-effortless-orchestration-759b9379cebe)
2. [Streamlining SonarQube on AWS ECS: Simplified Deployment Using Cluster.dev](https://aws.plainenglish.io/streamlining-sonarqube-on-aws-ecs-simplified-deployment-using-cluster-dev-0b988536fff0)

## Further reading

- [Getting started guide with EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html)
- [Cluster.dev documentation](https://docs.cluster.dev/)
- [What is Helm? A complete guide](https://circleci.com/blog/what-is-helm/)
- [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
