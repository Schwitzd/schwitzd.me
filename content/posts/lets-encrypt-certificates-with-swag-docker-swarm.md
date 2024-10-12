+++
title = 'Let's Encrypt Certificates with SWAG Docker Swarm'
date = 2024-10-07T19:26:02Z
draft = false
+++

Having worked out how to handle [TLS traffic on my K3S](https://www.schwitzd.me/posts/lets-encrypt-certificates-with-traefik/) setup, it is time to achieve the same goal on Docker. In this guide, I'll show you how to set up a Raspberry Pi running Docker Swarm, SWAG (Secure Web Application Gateway), and Let's Encrypt to secure your containerized applications with free TLS certificates. While I'll use Jellyfin as an example, this approach works for most containerized applications.

## Getting started

### Prerequisites

- A Raspberry Pi with Docker and Docker Swarm.
- A public domain managed, for example, by Cloudflare.
- Containerized applications, such as Jellyfin, running on Docker.
- SWAG to handle reverse proxy and TLS certificates.

### Folder structure

All my Docker containers data is stored on [BTRFS volume](https://www.schwitzd.me/posts/raspberry-btrfs/) under `/opt/my_pool` where I have created a folder for each container and the `nas-stack.yml` file with all the Docker instructions.

```sh
/mnt/my_pool/docker
├── jellyfin
├── nas-stack.yml
└── swag
```

## Docker Swarm

I opted for Docker Swarm instead of Docker Compose primarily for its better handling of secrets. With Docker Swarm, secrets are securely stored and only made available inside the container at runtime, mounted at `/run/secrets/<mysecret>`. This ensures that sensitive information, like passwords or API keys, is never exposed in clear text on the host.

In contrast, Docker Compose often requires bind-mounting secrets, which means they are stored in clear text on the host filesystem, making them more vulnerable if the host is compromised.

### Initiate the Docker cluster

To initiate a Docker Swarm on your system, use the following command:

```sh
docker swarm init
```

This command sets up the current machine as the Swarm manager.

### Create a Secret

To add a secret to your Docker Swarm, use the following command:

```sh
echo "your_secret_value" | docker secret create my_secret_name -
```

Personally, I don't like this approach because your secret is stored in your bash history, but unfortunately it's necessary because you can get trailing space problems with other methods. So I suggest you remove the line from your history with `history -d <line_number>`.
The other approach, which I will come to later, is to import the whole file as a secret:

```sh
docker secret create my_secret_name /path/to/your/secret_file
```

Again, the secret is in plain text on the file system, so if you don't need it, delete the file!

## SWAG

SWAG (Secure Web Application Gateway) is a Docker container that simplifies setting up a secure reverse proxy with automatic SSL/TLS certificates from Let's Encrypt. A key benefit of SWAG is that it comes with pre-built TLS configurations for many popular containerized applications, such as Jellyfin, Nextcloud, and more. This means you can quickly secure your applications without needing to manually configure HTTPS. SWAG also handles certificate renewal and provides a robust Nginx-based reverse proxy to ensure your web traffic is encrypted and secure.

I decided to include all my containers on the same stack and use a single overlay network for all of them.

### DNS validation and secrets

To get a TLS certificate from Let's Encrypt you need to prove that you own the domain, I use DNS challenge as a validation method ([more here](https://www.schwitzd.me/posts/lets-encrypt-certificates-with-traefik/#cloudflare-dns)), this implies that you need to store the API token in a configuration file to update the TXT record at your DNS provider (in my case Cloudflare).  

Each provider has its own configuration file, stored in `swag/config/dns-conf/`, this is the default configuration for Cloudflare:

```ini
# Instructions: https://github.com/certbot/certbot/blob/master/certbot-dns-cloudflare/certbot_dns_cloudflare/__init__.py#L20
# Replace with your values

# With global api key:
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234567

# With token (comment out both lines above and uncomment below):
#dns_cloudflare_api_token = 0123456789abcdef0123456789abcdef01234567
```

The strongly recommended authentication method is to use **API token** rather than **Global API key** as you can grant more granular permissions. In this case only **Edit: Zone.DNS** is required.  
As explained above, the secret will be in plain text on your host server, to avoid this I have decided to save the whole file as **docker secret** and delete it from the filesystem:

```sh
# create a Docker secret
docker secret cloudflare.ini swag/config/dns-conf/cloudflare.ini

# delete host file
rm swag/config/dns-conf/cloudflare.ini
```

We will reference the secret in the `nas-stack.yml` file with the following code:

```yaml
secrets:
  cloudflare.ini:
    external: true
```

### Reverse proxy

**SWAG** contains Nginx acting as a reverse proxy for managing incoming web traffic. The developers, to simplify our lives, have already prepared configuration files for popular apps and services such as Jellyfin. These configuration files are hosted on GitHub and are automatically pulled into the `/swag/config/nginx/proxy-confs` folder as inactive sample files (`jellyfin.subdomain.conf.sample`), ready to be customized and activated as needed.  
In my case the file works out of the box and we just need to rename it with the command `cp jellyfin.subdomain.conf.sample jellyfin.subdomain.conf`, but in some scenarios it is necessary to edit it according to your needs.

For example, in the Jellyfin configuration file you will find the following help:

```yml
# make sure that your jellyfin container is named jellyfin
# make sure that your dns has a cname set for jellyfin
# if jellyfin is running in bridge mode and the container is named "jellyfin", the below config should work as is
# if not, replace the line "set $upstream_app jellyfin;" with "set $upstream_app <containername>;"
# or "set $upstream_app <HOSTIP>;" for host mode, HOSTIP being the IP address of jellyfin
# in jellyfin settings, under "Advanced/Networking" add subdomain.mydomain.tld as a known proxy
```

### Deploy the stack

As mentioned earlier, the `nas-stack.yml` file contains all the instructions for deploying the Docker stack on the Swarm cluster. If we split it, we have:

- The `secrets` section, which contains the Cloudflare API token used by the SWAG service to generate TLS certificates.
- The `services` section, which defines the containers (such as Jellyfin and SWAG) with their configurations, such as images, volumes and ports.
- The overlay `networks`, where containers communicate internally and some services are exposed to the home network for external access.

```yaml
version: "3"

secrets:
  cloudflare.ini:
    external: true

services:
  swag:
    image: lscr.io/linuxserver/swag:latest
    cap_add:
      - NET_ADMIN
    hostname: swag-container
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - URL=example.com
      - VALIDATION=dns
      - SUBDOMAINS=jellyfin
      - CERTPROVIDER=
      - DNSPLUGIN=cloudflare
      - EMAIL=mail@example.com
      - ONLY_SUBDOMAINS=true
    secrets:
      - source: cloudflare.ini
        target: /config/dns-conf/cloudflare.ini
    volumes:
      - /mnt/my_pool/docker/swag/config:/config
    ports:
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    networks:
      - swarm-network
    deploy:
      mode: global
      labels:
        - com.centurylinklabs.watchtower.enable=true

  jellyfin:
    image: jellyfin/jellyfin
    hostname: jellyfin
    volumes:
      - /mnt/my_pool/docker/jellyfin/config:/config
      - /mnt/my_pool/docker/jellyfin/cache:/cache
      - type: bind
        source: /mnt/my_pool/media
        target: /media
    user: "1000:1000"
    networks:
      - swarm-network
    deploy:
      mode: global 

networks:
  swarm-network:
    driver: overlay
```

The **SWAG** container has some environment variables that need to be set to generate the TLS certificates:

- `URL` - The web address of the domain. This is used both for generating the TLS certificate and for DNS validation.
- `SUBDOMAINS` - List all the subdomains you want your TLS certificate to cover, or use wildcard to cover them all. This will result in a certificate for `*.example.com`.
- `VALIDATION` - Specifies the method the container will use to confirm your ownership of the domain. We will use DNS validation.
- `DNSPLUGIN` - Here I've selected Cloudflare as my DNS provider, which will handle the DNS validation.
- `EMAIL` - Enter your email address to receive your free Let's Encrypt SSL certificate for your domain.

The list of environment variables is available in the official [SWAG documentation](https://github.com/linuxserver/docker-swag/tree/master?tab=readme-ov-file#parameters).  

Also the Jellyfin container has its own configurations, but I will not cover them as it is outside the scope of this article, refer to the official documentation of the container application you need to use the appropriate settings.

The last step is to deploy the cluster and enjoy your container with a shiny free TLS certificate:

```sh
docker stack deploy -c nas-stack.yml nas
```

## Troubleshooting

Here are some useful command to troubleshoot your cluster environment if things goes wrong:

```sh
# Get stack services status
docker service ls

# Get container logs
docker service logs <service-name>
```

## Credits

I would like to thanks for inspiring me to write this article:

- [Setup SWAG to safely expose your self-hosted applications to the internet](https://blog.thelazyfox.xyz/setup-swag-to-safely-expose-your-self-hosted-applications-to-the-internet/)
