+++
title = 'Architecture'
date = 2024-05-04T12:23:10Z
draft = true
hidemeta = true
+++

This page provides a detailed overview of the architecture and implementation strategies behind my website. It delves into the fundamental principles guiding its design and development.

## High-Level architecture

My website is powered by [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static), using the [Hugo framework](https://gohugo.io/) for its structure and functionality.

<ins> Architecture component list: </ins>

* [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static)
* [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/)
* [Terraform](https://www.terraform.io/)
* [GitHub Actions](https://github.com/features/actions)

## Deploying infrastructure

Since I discovered [Terraform](https://www.terraform.io/) (yes, I know [IBM bought HashiCorp](https://www.hashicorp.com/blog/hashicorp-joins-ibm) and now people are switching to [OpenTofu](https://opentofu.org/)) I have become a big fan of [Infrastructure as Code](https://www.redhat.com/en/topics/automation/what-is-infrastructure-as-code-iac). I've opted to deploy all my Azure resources as code. Here's the [repository](https://github.com/Schwitzd/IaC-schwitzd.me) I utilize for managing these deployments.
