---
title: "I built a Kubernetes cluster on AWS EC2 with k3s"
date: 2026-11-16 00:00:00 -0700
categories: [Cloud, Kubernetes]
tags: [kubernetes, k3s, aws, ec2, networking]
---

I want to learn cloud skills, and the best way to understand Kubernetes is to build a cluster by hand. This is what I did, what each piece actually is, and where it broke.

## The plan

Two EC2 machines on AWS, turned into one Kubernetes cluster with **k3s**: one machine as the control plane (the "brain") and one as a worker. I chose k3s because it's real Kubernetes packed into a single small program, so I could learn the concepts without a long install.

## The building blocks

**AWS and EC2.** AWS rents out computers by the hour. **EC2** is its virtual machine service: each instance acts like a full computer with its own OS, CPU, memory, and disk. You choose an **AMI** (the OS template), an **instance type** (the hardware size), and a **key pair** (how you log in). Real companies run fleets of these, often created and destroyed automatically as traffic changes.

**VPC and security groups.** A **VPC** is a private network inside AWS, and a **security group** is a firewall attached to instances. It blocks everything unless a rule allows it. I set up three rules: SSH only from my own IP, all traffic between members of the group so my two nodes could talk freely without exposing their ports to the internet, and one port for the nginx page. Teams use the same pattern to keep databases and internal services off the public internet while still letting them talk to each other.

**Containers.** A container packages an app with everything it needs to run, in an isolated bubble that shares the host's operating system. It's lighter and faster to start than a full virtual machine, and it's why "works on my machine" mostly stops being an excuse: the app brings its environment with it.

**Kubernetes.** Running one container is easy. Running dozens across many machines raises problems: where should each one go, what happens when one crashes, how do you update without downtime? Kubernetes is the **orchestrator** that handles this. The core idea is **desired state**: I don't give it steps, I tell it what should be true ("3 copies of this app"), and it constantly compares reality to that description and fixes any gap. It's the same compare-and-correct loop I wrote about in my [harness post](/posts/agent-harness-loops-llm-ops/).

The vocabulary I picked up:

| Term | What it is |
|---|---|
| **Node** | One machine in the cluster |
| **Control plane** | The brain: an API server (front door), a state store, a scheduler (decides which node runs what), and controllers (the fix-it loops) |
| **Kubelet** | The agent on each worker that carries out the control plane's orders |
| **Pod** | The smallest unit, usually one container. Pods are disposable |
| **Deployment** | A rule like "keep 3 identical pods running" |
| **ReplicaSet** | What a Deployment actually manages: it counts pods and creates or deletes them to match the target |
| **Service** | A stable address in front of pods whose IPs keep changing |

## What I did

I launched two Ubuntu instances, connected over SSH using a key pair, and installed the k3s server on the first with its one-line installer. The installer sets k3s up as a background service so it survives reboots, and it also produces a **join token**, a shared secret that proves a new machine is allowed into the cluster. I then ran the installer on the second machine, pointed at the server's **private IP on port 6443** (the Kubernetes API port), which turned it into a worker. `kubectl get nodes` showed both machines `Ready`: one cluster made of two computers.

In real setups, nodes usually aren't joined by hand. Tools like Terraform and Auto Scaling groups run this same join step automatically on every new machine, so clusters grow and shrink on their own. Most companies also don't run the control plane themselves, since managed services like **EKS**, **AKS**, and **GKE** do that part.

Then I deployed nginx with a short YAML **manifest** describing two objects: a **Deployment** ("keep 3 copies running") and a **Service** that gives those pods one stable address and opens port 30080 on every node. The scheduler spread the pods across both machines, and `curl localhost:30080` on the server returned the nginx welcome page.

Next I scaled the Deployment from 3 to 5 replicas with one command. I never started anything myself, I only changed the target number, and the cluster did the rest. Here's a snapshot taken a second after scaling:

```text
NAME                   READY   STATUS              AGE   NODE
web-557577df75-28qjf   1/1     Running             28m   server
web-557577df75-42l4b   0/1     ContainerCreating   1s    agent
web-557577df75-chz5h   1/1     Running             28m   agent
web-557577df75-s6blf   0/1     ContainerCreating   1s    server
web-557577df75-w8cpw   1/1     Running             28m   agent
```

The new pods landed on different nodes, and the Deployment's event log recorded "scaled 3 to 5." The server runs pods too, because k3s lets its control-plane node run workloads by default (production clusters usually don't). Each pod gets its own IP from a separate range (`10.42.x.x`), and Flannel connects the ranges so pods on different machines can talk.

Then I tried the part I'd been curious about: I deleted a pod on purpose.

```text
$ sudo kubectl delete pod web-557577df75-28qjf
pod "web-557577df75-28qjf" deleted

NAME                   READY   STATUS    AGE
web-557577df75-42l4b   1/1     Running   6m34s
web-557577df75-8dxbt   1/1     Running   1s      <- new
web-557577df75-chz5h   1/1     Running   35m
web-557577df75-s6blf   1/1     Running   6m34s
web-557577df75-w8cpw   1/1     Running   35m
```

A new pod, with a different random suffix, was already running one second later, and I did nothing to create it. The Deployment wants 5 pods, the count dropped to 4, and its ReplicaSet created a replacement to close the gap. Kubernetes doesn't revive the old pod, it makes a fresh one from the template, which is why pods are treated as disposable and why Services exist to give them a stable address.

This self-healing is the main reason teams use Kubernetes. If a machine dies at 3 a.m., its pods get rescheduled onto healthy nodes before anyone wakes up. Production setups put a **load balancer** in front instead of a raw NodePort, but the Deployment-plus-Service pattern is the same.

Cloud resources bill until you delete them, so I terminated both instances (stopping only pauses compute charges, and the disks keep costing money).

## What I took away

- **Error messages narrow the problem.** "Permission denied" meant networking was fine, and "TLS timeout" meant the server was reachable but overloaded.
- **Kubernetes has real resource needs.** Checking memory and logs comes before suspecting the network.
- **This connects to my AI work.** Agents, eval pipelines, and tracing services are programs that need a reliable place to run. At a real job I'd more likely deploy onto an existing, managed cluster than build one, but now I know what's underneath.

## What I'd do next

1. **Rebuild it with Terraform.** Describe the instances, security group, and k3s install in code, so the whole cluster comes up from `terraform apply` and goes away with `terraform destroy`. That removes the copy-and-paste mistakes I made, and it's much closer to how teams actually work.
2. **Compare it with EKS.** The same nodes, pods, and Services, but with the control plane managed by AWS. I want to see what I'm no longer responsible for, and what that costs.