+++
title = "Common problems, and how to solve them with Kubernetes"
date = 2022-08-09
description = "A fictionalized account of how I adopted k8s"
draft = false

[taxonomies]
tags = ["kubernetes", "ops"]
+++

I first learned Kubernetes in 2018, when my manager sat me down and said "Cloudflare is migrating to Kubernetes, and you're handling our team's migration." This was slightly terrifying to me, because I was a good programmer and a mediocre engineer. I knew how to write code, but I didn't know how to deploy it, or monitor it in production. My computer science degree had taught me all about algorithms, data structures, type systems and operating systems. It had not taught me about containers, or ElasticSearch, or Kubernetes. I don't think I even wrote a single YAML file in my entire degree. I was scared of ops. I was _terrified_ of Kubernetes.

I read a lot of kubernetes introductions that year, and I developed a theory: by the time you're advanced enough to understand kubernetes, you're too advanced to properly explain it to a beginner. You might intend to write a simple, straightforward tutorial, but somehow the final product will come out totally incomprehensible to its target audience. It's the curse of inferential distance -- you've lost your beginners mind, and you can't talk in a way that beginners will understand.

Eventually I made it through and migrated all the Cloudflare Tunnel infrastructure from Marathon to Kubernetes. I didn't enjoy it, and I was way over my deadline, but I did learn a lot. Now it's 2022, and I'm leading a small team of engineers, some of whom have never used Kubernetes before. So I need to defy the curse of inferential distance, and explain Kubernetes to a beginner. 

In my opinion, the problem with Kubernetes articles is that they do everything backwards. They introduce k8s concepts without ever really addressing the problems you're trying to solve. They explain things from the perspective of a k8s maintainer, not of a normal software engineer trying to deploy their software to their work's k8s cluster, and running into problems. But this blog post is going to be a bit different. I'm going to tell you a story about a junior engineer. We're going to follow the problems they have, building a service in k8s, and how they solve them. This should introduce you to all the big basic ideas of Kubernetes and get you ready for building your own service in k8s.

<!-- more -->

# Introduction: why use k8s?

What problem is k8s even trying to solve? Why do we need another new complicated software system, with its own weird proper nouns and YAML schemas and certification process? I'll answer this, but before I do, we need to take a quick step back and rewind to 2013, where the software industry first became obsessed with:

## Containers

Say you want to deploy two Python servers. One of them needs Python 3.4 and the other needs Python 3.5. How do you deploy them on the same machine? You can hack a shell script together, maybe use Python-specific tools like [venv]. Fine, that works. But it doesn't scale very well, when you start deploying dozens of Python services, especially because you get similar problems with Node or Ruby or all the other languages you're using. 

So instead, you run each Python server in its own _container_. A container, like a Virtual Machine, lets your service act like it's the only service on this machine. It gets its own clean filesystem, and now you don't have to worry about clashes with other dependencies. Great! 

The big advantage containers have over VMs is that they're really lightweight. You can run _way_ more containers on a single machine than VMs. Why? Because each VM has to install its own OS, but containers can share the same underlying OS. For more beginner-level intro to containers, read [this][primer-on-containerization].

![Drawing of VMs with their own OS compared to containers, which share an OS](/kubernetes-problems/containers-vs-vms.png)

Perfect. You can now deploy all your services on the same machine, in complete isolation. There's no way they can interfere with each other. This is _awesome_ until you realize your services actually do need to talk to each other sometimes. Turns out letting processes message each other is a _pretty common thing and critical to your job_ -- maybe services need to query each other, or read from the same file, or one service needs to check the health of another service. And when you containerized everything, you just made that impossible.

## Container orchestration

Damnit! We have a problem. Using containers, we built each service a perfect little isolated cell, so they can't communicate. But we did our job too perfectly, we actually do need them to communicate. But only in specific ways that you, the programmer, choose. We don't want to go back to the pre-container days where all programs shared everything -- we saw how much of a mess that creates.

The other problem you'll face is _which containers should be running in the first place_. Your company might have a [Continuous Integration] pipeline which automatically builds a container every time you merge a PR to master. But now you've got all these hundreds or thousands of containers. How do you tell your servers which containers to run?

We solve this with a _container orchestration_ program, which selectively enables some containers and grants them some resources, including "communication with other containers". The phrase "container orchestration" is a metaphor for an orchestra. Imagine if no musicians in the orchestra could hear each other. They'd play at different speeds, the trumpets would drown out the oboes, every violinist would assume they were the soloist, and none of them would play the harmony parts. It would be chaos. To make this mess of isolated musicians into an orchestra, you need _structure_ and to enforce structure, you need a good conductor. 

A container orchestration program lets you:

 - Choose which containers should run
 - How many copies of each container should run
 - Which containers are allowed to share resources, like volumes or directories or files or networks
 - Choose which containers can access the same ports and hosts, to network with each other

Containers solve the problem of "how do I stop these services interfering with each other". Container orchestration solves "how do I let these services work together". They also solve "which services should be running". 

There are many different competing orchestration tools. You might have used [Docker Compose], it's simple and it's built into Docker, the most popular container engine. If you only need to run your containers on one physical machine, then Docker Compose is the orchestrator I'd suggest. But of course, today's software companies never deploy software to just one machine. Firstly, that single machine probably can't handle all your traffic. Secondly, even if it _could_, it would be a single point of failure -- if that machine goes down, your services become unavailable, and your customers can't place orders, and you lose money. So, real-world engineers need to orchestrate containers across many different physical servers. 

When I joined Cloudflare, we used [Apache Marathon] for this, but we were rapidly migrating to Kubernetes. This led to one big question:

## Why kubernetes?

Kubernetes exists to solve one problem: how do I run _m_ containers across _n_ servers? 

![A kubernetes cluster running n containers across m machines](/kubernetes-problems/k8s-m-n.png)

Its solution is a _cluster_. A Kubernetes cluster is an abstraction. It's a big abstract virtual computer, with its own virtual IP stack, networks, disk, RAM and CPU. It lets you deploy containers as if you were deploying them on one machine that didn't run anything else. Clusters abstract over the various physical machines that run the cluster. 

The cluster runs on your _n_ servers. A server which is part of a Kubernetes cluster is called a _node_.

A good abstraction should hide details from you, but let you access those details if you really need to. Generally your containers don't know and don't care which node they're running on -- the abstraction of the cluster is good enough. But if you really need to, you can query and inspect the various nodes that make up the cluster.

So, why use Kubernetes?

 - You want to use containers, to deploy your services in isolation from each other
 - You want to use a container orchestrator, to manage deployment of your containers and sharing resources between them
 - If you're running on one machine, just use Docker Compose
 - If you're running on many machines, which might come up and down, and need to schedule your containers across them, use Kubernetes.

# Solving problems with Kubernetes


---

[venv]: https://docs.python.org/3/library/venv.html
[Apache Marathon]: https://mesosphere.github.io/marathon/
[Docker Compose]: https://docs.docker.com/get-started/08_using_compose/
[primer-on-containerization]: https://increment.com/containers/primer-on-containerization/
[Continuous Integration]: https://martinfowler.com/articles/continuousIntegration.html