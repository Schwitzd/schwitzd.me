+++
title = 'Home K3s Cluster: My Journey Into Self-Hosting & Automation'
date = 2025-06-01T11:30:23Z
draft = false
+++

## Why I Did It

At first, this was a learning project. I wanted to understand the real nuts and bolts of Kubernetes — not just on paper, but on actual, bare-metal hardware in my home.

But over time, it became something more. It became my **platform** — the place where I host the things I care about, where I experiment, where I break things and fix them again.

## What I Wanted

- To **learn** how things really work under the hood  
- To **automate everything** — no clicking, no guesswork  
- To **host my own services** with security and flexibility in mind  
- To **power things down** when I don’t need them, and boot them up instantly when I do

## What I Built

The stack is simple but powerful:

- **K3s** – The lightweight Kubernetes friend of Raspeberry Pi
- **Terraform** – To define and control everything, from Helm charts to secrets
- **Ansible** – To configure and prepare the nodes exactly how I want them
- **Helm** – For reproducible, flexible application installs
- **Argo CD** – My GitOps engine: if it's not in Git, it doesn't exist

## The Digital Farm

Yes, I named the nodes after **farm animals**.  
Because why not? This is my playground.

- 🐮 **cow** – Raspberry Pi 5 (8GB RAM) – the control-plane  
- 🐑 **sheep** – Raspberry Pi 5 (16GB RAM) – general-purpose worker  
- 🦆 **duck** – Lenovo M910q (i7, 32GB RAM) – my AI muscle, mostly asleep until needed

They live on my **MikroTik-based home network**, which I also manage with Terraform.  
You can read about my networking automation journey in this [blog post](/posts/mikrotik-terraform-1/).

## On Power, Silence & Grace

One of the most satisfying parts of this cluster is its **graceful shutdown and startup workflow**.

When I don’t need it, I power it down — safely:

- Scale down all Longhorn workloads
- Drain the nodes
- Stop k3s
- Power off

When I bring it back up, a `systemd` service uncordons the nodes, and Argo CD takes care of the rest.

No corrupted volumes. No drama.

## What's Coming

This post is just an overview. In the coming weeks, I'll publish deeper dives into how I configured specific workloads and the challenges I faced.

If I've sparked your interest in self-hosting, automation, or building cool things, feel free to explore the full project on GitHub — everything is fully declarative, version-controlled.

👉 [github.com/Schwitzd/IaC-HomeK3s](https://github.com/Schwitzd/IaC-HomeK3s)
