+++
title = 'Architecture'
date = 2024-05-04T12:23:10Z
draft = true
hidemeta = true
+++

This page provides a detailed overview of the architecture and implementation strategies behind my website. It delves into the fundamental principles guiding its design and development.

## Principles

* Both the infrastructure and the content should be stored on my [Github](https://github.com/Schwitzd)
* Provision as much as possible as code
* Don't use traditional hosting services and CMSs
* DRY (Don't Repeat Youself)

## High-Level architecture

My website is powered by [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static), using the [Hugo framework](https://gohugo.io/) for its structure and functionality.

![Architecture Schema](/img/architecture.png)

Architecture component list:

* [Azure Static Web App](https://azure.microsoft.com/en-us/products/app-service/static)
* [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/)
* [Terraform](https://www.terraform.io/)
* [GitHub Actions](https://github.com/features/actions)

## Deploying infrastructure

Since I discovered [Terraform](https://www.terraform.io/) (yes, I know [IBM bought HashiCorp](https://www.hashicorp.com/blog/hashicorp-joins-ibm) and now people are switching to [OpenTofu](https://opentofu.org/)) I have become a big fan of [Infrastructure as Code](https://www.redhat.com/en/topics/automation/what-is-infrastructure-as-code-iac). I've opted to deploy all my resources as code.  
Here's the [repository](https://github.com/Schwitzd/IaC-schwitzd.me) I use to manage these deployments, with all the detailed steps I took to provision the infrastructure.

To summarise:

1. Bought my own domain on Cloudflare.
1. Provided DNS records for my root and 'www' subdomain.
1. Provisioned the Azure Static App
1. Added the DNS TXT record to validate that I'm the owner of the domain
1. Deploy the Github action to automatically push changes to the Azure Static App

## Design Walkthrough

* Most of my changes are documented on the [changelog](/changelog) page
