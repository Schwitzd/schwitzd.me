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

## Cloudflare

By default Cloudflare configures [TLS encryption mode](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/) to **Flexible**, from the official documentation: *Cloudflare allows HTTPS connections between your visitor and Cloudflare, but all connections between Cloudflare and your origin are made through HTTP*.  
At the same time, Azure enforces HTTPS redirection and this leads to the browser error [ERR_TOO_MANY_REDIRECTS](https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/). To resolve this issue, go to your Cloudflare Dashboard → SSL/TLS → Overview and select **Full (strict)**.

## Configure Azure Static Web Apps

I have defined a [custom configuration](https://learn.microsoft.com/en-us/azure/static-web-apps/configuration) for Azure Static Web Apps with the [staticwebapp.config.json](/staticwebapp.config.json) created in the *static* folder.

### Security headers

Some will laugh, others will think it's overkill.. but the main reason I added all these security headers is for learning purposes. The site is now rated **A** on [securityheaders.com](https://securityheaders.com/?q=https%3A%2F%2Fwww.schwitzd.me%2F)

Why not **A+**? Because at the time of writing PaperMod uses inline Javascript and styles. I have reported an issue [CSP Enhancement by removing unsafe-inline #1517](https://github.com/adityatelange/hugo-PaperMod/issues/1517), but it does not seem to be a priority and my impression is that it will take a lot of time.

## Development

As explained on the [How I Set Up This Site](posts/how-i-set-up-this-site/) page, I use [Docker](https://www.docker.com/) to host this site during the development, with the [docker-hugo](https://github.com/Schwitzd/docker-hugo) image I created to suit my needs.
I added the [Hugo server configuration file](https://gohugo.io/getting-started/configuration/#configure-server), which is useful for testing the [CSP policy](#security-headers) before deploying the site to Azure.

```bash
touch config/development/server.yaml
```
