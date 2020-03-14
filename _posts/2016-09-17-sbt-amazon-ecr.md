---
layout: post
title: "Publishing a Docker image to Amazon ECR using SBT"
excerpt: "Guidelines for deploying a Scala application to an Amazon ECR repository."
tags: [amazon, aws, sbt, ecr, docker, cloud, scala]
modified: 2016-09-17
comments: true
---

If you have a **Docker** infrastructure in **AWS**, then **Amazon ECR** is
a likely registry option being used in your environment. In this post I'll show you how to build a Docker image from a simple **Scala** application using the [sbt-native-packager](https://github.com/sbt/sbt-native-packager) plugin, and how to publish it to Amazon ECR using [sbt-ecr](https://github.com/sjednac/sbt-ecr).

Once published, the image could be deployed by **Amazon ECS** or any other service, that
can authenticate to **Amazon ECR** and pull Docker images from it. Note that service deployment isn't covered by the scope of this article, just the *getting my artifact to some repository in an Amazon region* part.

## Hello World in Docker

Consider a minimal **Scala** application:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 SbtAmazonEcr.scala %}

... with a simple build config in `build.sbt`

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 build.sbt %}

We can add **Docker** capabilities to this SBT project by including the
**sbt-native-packager** plugin in the `project/plugins.sbt` file:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 project-plugins.sbt %}

... and enabling some additional settings for the Docker build itself:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 docker.sbt %}

Once set up, you can run `sbt docker:publishLocal` to build the Docker image and
publish it to the local registry.

## Publishing to Amazon ECR

Add the [ECR plugin](https://github.com/sjednac/sbt-ecr) to your `project/plugins.sbt`:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 project-plugins-ecr.sbt %}

Include the following project settings:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 ecr.sbt %}

Once everything is set up you can `push` the image to ECR by running:

{% gist sjednac/439fa37054c1cec326a10ddcd823fae0 push.sh %}

Please note, that you may need to set up [credentials](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files) for the **Amazon SDK**, if you haven't done it earlier.
