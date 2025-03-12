# Media Server Example - Plex & \*arr Apps on Docker Compose

## Description

This repo has misc config files and instructions for setting up a Docker/Plex/arr-based media server. It includes step-by-step instructions for setting up the environment from scratch.

I'm completely new to all of this and when I first decided to try it out I had to look a million different places to figure everything out. This is meant to be a place to document all setup steps from 0 to running server.

More specifically this repo contains:

- docker-compose.yml file containing all the software/services required
- .env file containing values referenced by docker-compose. Things like file paths, URLs and domain names, API keys, etc.
- homepage config. Homepage is a third-party application dashboard that can display links to all the docker services in one place
- Instructions in this readme for setting up the environment

## Table of Contents

<!-- TOC -->

- [Media Server Example](#description)
  - [Table of Contents](#table-of-contents)
  - [Disclaimers](#disclaimers)
  - [Technology Stack & Tools](#technology-stack--tools)
    - [Routing](#routing)
    - [SSL Certificates](#ssl-certificates)
    - [Internal vs External Access](#internal-vs-external-access)
  - [Application URLs](#application-urls)
  - [File System](#file-system)
  - [Docker Compose File Overview](#docker-compose-file-overview)
    - [Labels for Non-Traefik Containers](#labels-for-non-traefik-containers)
      - [Pullio-Related Labels](#pullio-related-labels)
      - [Traefik-Related Labels](#traefik-related-labels)
    - [Environment Variables For Non-Traefik Containers](#environment-variables-for-non-traefik-containers)
    - [Labels, Environment Variables and Commands for Traefik](#labels-environment-variables-and-commands-for-traefik)
      - [Traefik Commands](#traefik-commands)
      - [Traefik Environment Variables](#traefik-environment-variables)
      - [Traefik Labels](#traefik-labels)
  - [Server Setup Steps](#server-setup-steps)
    - [Server Setup](#server-setup)
    - [Docker Setup](#docker-setup)
    - [Post-Install Application Setup](#post-install-application-setup)
  - [Misc](#misc)
    - [Helpful Docker Compose Commands](#helpful-docker-compose-commands)
  - [Credits](#credits)
  <!-- TOC -->

## Disclaimers

- I'm writing these instructions in retrospect after setting everything up, so it's likely some setup steps are out of order or have been missed entirely, specifically with application-specific configuration after installs. I followed trash-guides heavily, specifically these sections:
  - [How To Set Up > Docker](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/)
  - [How To Set Up > Synology](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Synology/)
  - Application-specific setup like [radarr](https://trash-guides.info/Radarr/)
- I'm not a sysadmin/network/security person, so I don't claim this is the most secure way of doing things. There could even be vulnerabilities in this setup. I did the best I could lock things down reasonably well.
- Plex, media streaming, etc. is completely new to me. I'm not an expert on codecs, media quality, compatibility with players, etc. I just have a smart TV I wanted to stream some of my DVDs to so I didn't put much effort into optimizing the actual quality profiles and custom formats.
- I use this only for usenet, I don't know anything about torrents. This solution could easily be adapted for them I assume, but I don't know how.

## Technology Stack & Tools

- [Ubuntu Server 24.04.2 LTS](https://ubuntu.com/download/server)
- [Docker (Docker compose)](https://docs.docker.com/compose/)
  - This solution does not use docker desktop, portainer, etc. Purely docker compose CLI commands
- [Plex](https://plex.tv)
- \*arr apps
  - [Radarr](https://radarr.video/)
  - [Sonarr](https://sonarr.tv/)
  - [Prowlarr](https://prowlarr.com/)
  - [Profilarr](https://dictionarry.dev/)
- Usenet clients/providers
  - [NZBGeek](https://nzbgeek.info/)
  - [NZBGet](https://nzbget.net/)
  - [Frugal Usenet](https://frugalusenet.com/) or whatever provider(s) you have
- [Traefik](https://traefik.io/traefik/) reverse proxy
  - [Let's encrypt](https://letsencrypt.org/) for SSL
- [Cloudflare domain/DNS](https://www.cloudflare.com)
- [Pullio](https://hotio.dev/scripts/pullio/)
- [Homepage Dashboard](https://gethomepage.dev/)

This is something I setup on an old PC I wasn't using. It doesn't involve a NAS or anything like that, everything runs on a Linux server with internal hard drives. It wouldn't be hard to extend this solution to a NAS, or maybe even run the entire thing on one.

At a high level, all apps run in their own docker container. Traefik is used as a reverse proxy such that only ports 80 and 443 are exposed in your docker-compose.yml file. Internal ports for apps like radarr, sonarr, etc. are not exposed.

### Routing

I prefer subdomains versus URL paths to reach applications. For example I prefer sonarr.mydomain.com over mydomain.com/sonarr. It's not more effort, and you don't need to modify application-specific configuration in sonarr/radarr to have them respond on that path. Therefore Traefik is configured to recognize routes based on subdomain and not path. A wildcard DNS record for \*.mydomain.com is used in Cloudflare, but you could use hosts file entries or some internal DNS like pihole instead. Using a wildcard entry in Cloudflare means we don't expose service names in our public DNS records like sonarr.mydomain.com. There may be reasons why subdomains shouldn't be used, but I don't have any other servers or domains to manage, so for now it works for me.

### SSL Certificates

This setup supports SSL certs through Cloudflare with a custom domain, but that is not required. Cloudflare sells domains at cost and otherwise has free DNS services.

A DNS-01 chaallenge is used with Let's Encrypt to generate certificates. Traefik supports this automatically with Cloudflare. It does support other DNS providers as well, they can be viewed on the [Traefik docs site](https://doc.traefik.io/traefik/https/acme/#providers). Setting up alternative providers is outside the scope of this guide, but you would need to modify your .env and docker-compose.yml file to do so

### Internal vs External Access

Because of the way the DNS records are configured in this solution the environment can only be accessed from you LAN. This setup is very close to allowing external access but I haven't done that yet so it's not in the guide.

At a high level, external access can be achieved either by enabling a VPN, or port forwarding ports 80/43 in your router directly to Traefik. You also would need a solution for dynamic DNS, which can be implemented easily with Cloudflare's API and a cron script. However extra security concerns need to be considered when enabling external access.

## Application URLs

After setup is complete you will be able to access your software at the following URLs:

| App                   | URL                                     | Description                                     |
| --------------------- | --------------------------------------- | ----------------------------------------------- |
| NZBGet                | https://nzbget.mydomain.com             | Usenet downloader                               |
| Prowlarr              | https://prowlarr.mydomain.com           | Usenet indexer management for Radarr and Sonarr |
| Sonarr                | https://sonarr.mydomain.com             | PVR for Usenet, i.e. manage TV series           |
| Radarr                | https://radarr.mydomain.com             | Movie collection manager for Radarr             |
| Profilarr/Dictionarry | https://profilarr.mydomain.com          | Quality profile manager for Sonarr/Radarr       |
| Traefik Dashboard     | https://traefik.mydomain.com/dashboard/ | Reverse proxy                                   |
| Plex                  | https://plex.mydomain.com               | Media streaming server                          |
| Homepage Dashboard    | https://mydomain.com                    | Customizable application dashboard              |

## File System

I use the TRaSH Guides [recommended file and folder structure](https://trash-guides.info/File-and-Folder-Structure/). The main benefits of this approach are hardlinks and instant moves which are explained more in the TRaSH Guides recommendation page. The key is that all storage for downloaders, media library, etc. are on the same file system. That way all your containers can have access to the folders they need but see the file system structure in the same way. I'm using a volume I mounted at /hdd, and at the end of this setup this is what the file system will look like:

         hdd
         ├── data
         │   ├── media
         │   │   ├── movies
         │   │   ├── music
         │   │   └── tv
         │   └── usenet
         │       ├── complete
         │       │   ├── movies
         │       │   ├── music
         │       │   └── tv
         │       ├── incomplete
         │       │   ├── movies
         │       │   ├── music
         │       │   └── tv
         └── docker
            └── appdata
                ├── bazarr
                ├── homepage
                ├── letsencrypt
                ├── nzbget
                ├── plex
                ├── profilarr
                ├── prowlarr
                ├── pullio
                └── radarr

- /hdd/docker/appdata is for a all of your docker containers' configuration databases, logs, etc. It supports the applications themselves.
- /hdd/data/usenet is for your downloaders
- /hdd/data/media is for your actual media library used by Plex, \*arr apps, etc.

## Docker Compose File Overview

The docker-compose.yml file in this repo has containers for all the applications above. For the most part this file should not need to be edited. Most of the environment variables, volumes, labels, etc. follow a very similar pattern for all containers which is explained below. The Traefik container is a special case so that is explained separately

### Labels For Non-Traefik Containers

#### Pullio-Related Labels

All containers have three Pullio-related labels. Pullio is a bash script scheduled via a cron job that updates your docker containers automatically. More info can be found [here](https://hotio.dev/scripts/pullio/).
| Label | Description
| --- | --- |
| org.hotio.pullio.update | true/false. If set to false Pullio will ignore this container and won't update it |
| org.hotio.pullio.notify | true/false. if set to false Pullio will not send any notifications when updates occur. If true, discord notifications are supported |
| org.hotio.pullio.discord.webhook | Populate with a webhook created in your Discord channel. Only Discord or custom webhooks are supported |

#### Traefik-Related Labels

By default, Traefik looks for container-level labels to configure how requests should be routed. It supports way more than is being used here, so there are only a few simple labels for each container. The Traefik container itself is a special case and will be explained separately.

You will see labels named traefik.http.routers.XXXX.someSetting and traefik.http.services.XXXX.someSetting. Each service needs its own configuration, and the XXXX is a somwhat arbitrary label used to define that site's specific configuration. It can be any unique value, but by convention it's named the same as the container.

| Label                                                                        | Description                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| traefik.enable                                                               | True/false. Should always be true. If set to false traefik will ignore this container and your application won't be accessible                                                                                                                               |
| traefik.http.routers.radarr.rule=(Host(\`radarr.${SERVICE_BASE_HOSTNAME}\`)) | Used to match against a hostname for routing traffic to the container. Each container uses the SERVICE_BASE_HOSTNAME .env variable as the base domain. In this example Traefik will send traffic to this container if the domain/host = radarr.mydomain.com. |
| traefik.http.routers.radarr.tls.certresolver=myresolver                      | myresolver is defined inside the traefik container itself, and has settings for how to provision SSL certificates. This label on each container is a pointer to that resolver. In this setup they can all use the same resolver (Let's Encrypt/Cloudflare)   |
| traefik.http.services.radarr.loadbalancer.server.port=7878                   | This should be set to the port on which the application in that container responds. All the \*ar apps, NZBGet, etc. respond on a different port by default. The values in this compose file all use the default ports for their respective applications      |

### Environment Variables For Non-Traefik Containers

Just about all containers have 4 main environment variables, with a couple extra for plex:

| environment: variable | Description                                                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| PUID                  | User under which docker container processes will run                                                                               |
| PGID                  | Group under which docker container processes will run                                                                              |
| TZ                    | Timezone, may not work depending on docker image you're using. I don't know if these specific images have tzdata package installed |
| UMASK                 | Sets file creation permissions. 002 was used in the examples I found and frankly that's why you see it here                        |
| PLEX_CLAIM_TOKEN      | Used in plex image, needed to register your plex server                                                                            |
| PLEX_ADVERTISE_URL    | IP/Port of your Plex server on your LAN. Used to publish your media server's address for discovery                                 |
| PLEX_NO_AUTH_NETWORKS | List of IP addresses and networks that are allowed without auth                                                                    |
| PLEX_BETA_INSTALL     | True/false, whether beta builds of plex are allowed. Requires plex pass                                                            |

### Labels, Environment Variables, and Commands for Traefik

Traefik is a reverse proxy that helps secure your environment (more is needed to fully secure it) and access your apps with friendly URLs. There two important concepts these labels, variables, and comments relate to:

SSL Certificate provisioning: Traefik uses Let's encrypt to automatically provision SSL certificates for you, specifically using the [DNS-01 challenge type](https://doc.traefik.io/traefik/https/acme/#dnschallenge). Traefik communicates with Let's encrypt to get a token. It then uses the Cloudflare API to create a DNS record with that token. Let's encrypt sees the token, trusts that you own the domain, and issues the certificate. Many of the commands and environment variables are related to configuring this process.

Routing: The other configuration items are related to basic routing like automatic redirect from HTTP to HtTPs, automatic discovery of your docker containers, access to the traefik dashboard interface, etc.

#### Traefik Commands

| Command                                                                                                                                                                                                                                                                         | Description                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ping                                                                                                                                                                                                                                                                            | True/false. Enabled health check pings for Traefik instance                                                                                                                                               |
| api.dashboard                                                                                                                                                                                                                                                                   | True/false, enable Traefik dashboard access or not                                                                                                                                                        |
| providers.docker                                                                                                                                                                                                                                                                | True/false, should always be true. Enables the Traefik Docker integration to auto-discover your other containers. Thi works in tandem with the traefik-related labels you supply in individual containers |
| providers.docker.exposedbydefault                                                                                                                                                                                                                                               | True/false. If set to true Traefik will enable routing for all containers by default. If set to false, each container must have a traefik.enable=true for it to be exposed/enabled in Traefik.            |
| entrypoints.web.address <br/> entrypoints.web-secure.address <br/> entrypoints.web-secure.http.tls <br/> entrypoints.web.http.redirections.entryPoint.to <br/> entrypoints.web.http.redirections.entryPoint.scheme <br/> entrypoints.web.http.redirections.entrypoint.permanent | These commands together define a web entry point listening on port 80, a web-secure entry point listening on port 443, and requests to port 80 will get redirected to port 443                            |
| certificatesresolvers.myresolver.acme.dnschallenge                                                                                                                                                                                                                              | True/false, tells myresolver to use a dnschallenge if true. Should always be true with current configuration                                                                                              |
| certificatesresolvers.myresolver.acme.dnschallenge.provider                                                                                                                                                                                                                     | Tells traefik which DNS provider to use, in this case Cloduflare                                                                                                                                          |
| certificatesresolvers.myresolver.acme.dnschallenge.resolvers                                                                                                                                                                                                                    | DNS servers to use to resolve FQDN authority                                                                                                                                                              |
| certificatesresolvers.myresolver.acme.caserver                                                                                                                                                                                                                                  | Let's encrypt server to use                                                                                                                                                                               |
| certificatesresolvers.myresolver.acme.email                                                                                                                                                                                                                                     | Email reported to Let's encrypt for certificates and renewals                                                                                                                                             |
| certificatesresolvers.myresolver.acme.storage                                                                                                                                                                                                                                   | Location to cert file. The /letsencrypt volume is mapped to this container in the volumes config                                                                                                          |

#### Traefik Environment Variables

| environment: Variable     | Description                                                                                    |
| ------------------------- | ---------------------------------------------------------------------------------------------- |
| CLOUDFLARE_EMAIL          | Email associated with your Cloudflare account                                                  |
| CLOUDFLARE_DNS_API_TOKEN  | Traefik uses these API tokens to read and edit your DNS for the Let's Encrypt DNS-01 challenge |
| CLOUDFLARE_ZONE_API_TOKEN | Traefik uses these API tokens to read and edit your DNS for the Let's Encrypt DNS-01 challenge |
| LETS_ENCRYPT_EMAIL        | Email for Let's Encrypt certificate                                                            |

#### Traefik Labels

| Label                                           | Description                                                                                                                                                                                                                                            |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| traefik.http.routers.dashboard.rule             | This serves the same purpose as the other rules, but is slightly different. Traefik exposes the dashboard on /dashboard/ URL, which also requires /api/. We only want requests to get to the Traefik container unless they're for the dashboard or api |
| traefik.http.routers.dashboard.service          | Attach this router to the api@internal service, which then allows you to secure the dashboard with router rules, basic HTTP auth, etc. I use basic HTTP auth                                                                                           |
| traefik.http.routers.dashboard.middlewares      | Used to wire up middleware for this router's request pipeline. In this case, used to add a myauth middleware which is basic HTTP auth                                                                                                                  |
| traefik.http.middlewares.myauth.basicauth.users | Enables basic auth and sets the list of allowed users to a user you generate later in these steps                                                                                                                                                      |

## Server Setup Steps

This section is a long series of steps, from installing Ubuntu Server on a blank machine to having running applications. It follows three major areas:

- [Server setup](#server-setup)
- [Docker setup](#docker-setup)
- [Post-install application setup](#post-install-application-setup)

If you already have a server running you could skip the first section.

### Server Setup

1. We want to manage this server remotely via SSH. That could be done with the user/name password created during Ubuntu install, but it can also be done via SSH keys. SSH keys are better for various reasons. If you don't want to use SSH keys, continue to install ubuntu server step.
   1. [Generate new SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) and add it to the ssh-agent.
   1. [Add the SSH key to your github account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
1. Install Ubuntu Server LTS on host machine. Make sure to have local monitor and keyboard, and it's easiest to plug ethernet cable directly into server. If you generated github SSH keys, import them during install.
1. Install all available Ubuntu updates
   ```bash
   sudo apt update
   sudo apt dist-upgrade
   sudo apt upgrade
   ```
1. Enable auto-updates
   ```bash
   sudo apt install unattended-upgrades
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```
1. Install ubuntu-restricted-extras for media codecs and misc stuff
   ```bash
   sudo apt install ubuntu-restricted-extras
   ```
1. Give server a hostname and edit hosts file

   1. Set server hostname. If you plan on setting up a domain name, probably good to use that

   ```bash
   hostnamectl set-hostname whatever-my-server-name-is

   ```

   2. Update /etc/hosts file. Add record for 127.0.1.1 whatever-my-server-name-is

   ```bash
   sudo nano /etc/hosts
   ```

1. [Install docker engine](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

   1. Set up Docker's apt repository

   ```bash
   sudo apt-get update
   sudo apt-get install ca-certificates curl
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc

   echo \
   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
   $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   ```

   1. Install the Docker packages

   ```bash
   sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

1. Mount new volume for you media and appconfig directories which will be created later. It should be a single volume to support hardlinks as described in [trash-guides](https://trash-guides.info/File-and-Folder-Structure/Hardlinks-and-Instant-Moves/).
   - The PC I re-used had a 256GB SSD and a smaller HDD. i installed the OS to the SSD and mounted the HDD for my data.
   - Make sure to add the volume to /etc/fstab so it persists after reboot
1. Note IP of new server on your LAN
   ```bash
   hostname -I
   ```
1. Reserve this private IP for your server in your LAN
   - The process for doing so is outside the scope of this guide. It differs depending on what type of router/hardware you have in front of this server.
1. Reboot
   ```bash
   sudo reboot
   ```
1. Confirm SSH access to server.
   - On a different machine, run the following command to start SSH session. Replace with your username and IP from previous step
   ```bash
   ssh user@192.1681.21
   ```
1. Disable password access - Do not do this if you didn't generate SSH keys!
   1. Open sshd_config in nano
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   1. Set PermitRootLogin = no
   1. Set PasswordAuthentication = no
   1. Save File
   1. Restart sshd
   ```bash
   sudo systemctl restart sshd
   ```
1. Configure ufw firewall
   1. Basic setup - this disallows all external incoming traffic by default except for that on your LAN. You may need to update the 192.168.1.0/24 based on your environment. External access and more advanced setup is out of scope of this guide. At the very least internal traffic should still be locked down to specific ports like 80/443
   ```bash
   sudo ufw enable
   sudo ufw allow ssh
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow from 192.168.1.0/24 to any
   ```
1. Reboot
   ```bash
   sudo reboot
   ```
1. Set up folder structure from [trash guides](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Synology/#if-you-use-usenet). For the purposes of this guide, this folder structure will be created on a volume mounted at /hdd

   - For movie/data files:

   ```bash
   mkdir -p /hdd/data/{usenet/{incomplete,complete}/{tv,movies,music},media/{tv,movies,music}}
   ```

   - For docker app data:

   ```bash
   mkdir -p /hdd/docker/appdata/{radarr,sonarr,bazarr,plex,prowlarr,pullio,homepage,letsencrypt,nzbget,profilarr}
   ```

1. Set permissions on newly created folder structure. Replace adminUserName with the username you created during Ubuntu installation.
   ```bash
   sudo chown -R adminUserName:users /hdd/data /hdd/docker
   sudo chmod -R a=,a+rX,u+w,g+w /hdd/data /hdd/docker
   ```

### Docker Setup

1. Register a new domain or transfer your domain to Cloudflare. Set up your domain here, and take note of some API keys for later use.

   1. Set up DNS records for mydomain.com and \*.mydomain.com to point to the internal IP you noted earlier.
   1. Create two API tokens on your Cloudflare [profile page](https://dash.cloudflare.com/profile/api-tokens). These are used for the SSL cert DNS challenge.
      - One token should have Zone>DNS:Edit permissions. Note this for later use as CLOUDFLARE_DNS_API_TOKEN.
      - The other token should have Zone>Zone:Read permissions. Note this for later use as CLOUDFLARE_ZONE_API_TOKEN.

1. Create password for Traefik dashboard.
   - The Traefik dashboard should be secured in production use. This solution uses basic HTTP authentication. Traefik supports configuring credentials for specific users with hashed passwords. Generate a password for your username. This username and password will be used to log into the Traefik dashboard web interface. Note the user you use in this command as TRAEFIK_DASHBOARD_USER, and the password as TRAEFIK_DASHBOARD_HASHEDPASS.
   ```bash
   htpasswd -nb user password
   ```
1. Note the uid and gid returned from the following command as PUID and PGID respectively. Replace adminUserName with the admin user you created when installing Ubuntu server.
   ```bash
   id adminUserName
   ```
1. Create an account on Plex if you don't already have one. Go to [plex.tv/claim](https://plex.tv/claim) to claim your Plex server code. Note it as PLEX_CLAIM_TOKEN.
1. Download the .env file from this repo and update it. There are comments in the file already, but here is an overview based on the information you gathered in previous steps
   | .env Key | Notes |
   | -------- | ------|
   | COMPOSE_PROJECT_NAME | Arbitrary name to identify compose project |
   | DOCKERSTORAGEDIR | Base path to the data/media library you set up per [trash-guides recommendations](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Synology/#create-folder-structure) |
   | DOCKERCONFDIR | Base path to the app config folders you set up per [trash-guides recommendations](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Synology/#appdata) |
   | TZ | Timezone of your container, [valid values here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) |
   | PUID | uid from previous step |
   | PGID | gid from previous step |
   | SERVICE_BASE_HOSTNAME | custom domain you registered in Cloudflare, e.g. mywebsite.com |
   | TRAEFIK_DASHBOARD_USER | traefik username from previous step |
   | TRAEFIK_DASHBOARD_HASHEDPASS | traefik password from previous step |
   | CLOUDFLARE_EMAIL | Email of your Cloudflare account |
   | CLOUDFLARE_DNS_API_TOKEN | DNS token from previous step |
   | CLOUDFLARE_ZONE_API_TOKEN | Zone token from previous step |
   | LETS_ENCRYPT_EMAIL | Email let's encrypt uses for certs and notifications |
   | PLEX_CLAIM_TOKEN | Claim token from previous step |
   | PLEX_ADVERTISE_URL | Replace the IP with the IP of your server noted in previous step |
   | PULLIO_DISCORD_WEBHOOK | Create a webhook in your Discord channel if you want updates from Pullio when containers are updated |
1. Copy the .env and docker-compose.yml out to the root of your docker appdata directory (per trash guides folder structure). For example /hdd/data/docker
1. Download the /docker/appdata/homepage folder from this repo. Update the services.yaml file.
   - Replace the href: lines with the custom domain you registered in Cloudflare.
   - In the sonarr, radarr, and prowlarr sections, replace the key: lines with the API keys for each application. You can get them by going to Settings > General in each application.
   - Optionally update bookmarks.yaml, settings.yaml, or widgets.yaml. Additional settings files are available as well. Configuration of this app is out of scope, but more details can be found [here](https://gethomepage.dev/configs/)
1. Copy the entire contents of the /docker/appdata/homepage folder you modified and copy it to your server at /hdd/docker/appdata/homepage
1. Start Docker containers
   ```bash
   cd /hdd/docker/appdata
   sudo docker compose up -d
   ```
   - After starting the docker containers, it may take a minute or two for Traefik to discover all your services and perform the Let's Encrypt SSL Cert process. Once the docker containers are running you should be able to access your applications here:
     | App | URL |
     | --- | --- |
     | NZBGet | https://nzbget.mydomain.com |
     | Prowlarr | https://prowlarr.mydomain.com |
     | Sonarr | https://sonarr.mydomain.com |
     | Radarr | https://radarr.mydomain.com |
     | Profilarr/Dictionarry | https://profilarr.mydomain.com |
     | Traefik Dashboard | https://traefik.mydomain.com/dashboard/ |
     | Plex | https://plex.mydomain.com |
     | Homepage Dashboard | https://mydomain.com |

### Post-Install Application Setup

1. Initial NZBGet setup

   1. Navigate to https://nzbget.mydomain.com
   1. Set up the app based on [trash-guides setup](https://trash-guides.info/Downloaders/NZBGet/Basic-Setup/)
   1. Make sure to create a RestrictedUsername and RestrictedPassword under Settings > Security. We'll use this later.

1. Initial Sonarr setup

   1. Navigate to https://sonarr.mydomain.com
   1. Set up naming scheme for series per [trash-guides recommendations](https://trash-guides.info/Sonarr/Sonarr-recommended-naming-scheme/)
   1. Navigate to Settings > Media Management and click show advanced.
   1. Under File Management change Propers and Repacks to Do not Prefer, per [ictionarry/Profilarr recommendations](https://dictionarry.dev/wiki/profilarr-setup)
   1. Under Settings > Quality, change all the min slider values to 1 and the max slider values to 1000 per [ictionarry/Profilarr recommendations](https://dictionarry.dev/wiki/profilarr-setup)
   1. Under Series > Library Import, add your media folder, e.g. /data/media/tv
   1. Under Settings > Download Clients, add your NZBGet download client. Use the following settings:
      - Host: nzbget
      - Port: 6789
      - Username: Restricated access username from previous steps
      - Password: Restricted access password from previous steps
      - Category: tv
   1. Under Settings > General take note of the API key, it will be needed later.

1. Initial Radarr setup

   1. Navigate to https://radarr.mydomain.com
   1. Set up naming scheme for movies per [trash-guides recommendations](https://trash-guides.info/Radarr/Radarr-recommended-naming-scheme/)
   1. Navigate to Settings > Media Management and click show advanced.
   1. Under File Management change Propers and Repacks to Do not Prefer, per [ictionarry/Profilarr recommendations](https://dictionarry.dev/wiki/profilarr-setup)
   1. Under Settings > Quality, change all the min slider values to 1 and the max slider values to 1000 per [ictionarry/Profilarr recommendations](https://dictionarry.dev/wiki/profilarr-setup)
   1. Under Movies > Library Import, add your media folder, e.g. /data/media/movies
   1. Under Settings > Download Clients, add your NZBGet download client. Use the following settings:
      - Host: nzbget
      - Port: 6789
      - Username: Restricated access username from previous steps
      - Password: Restricted access password from previous steps
      - Category: movies
   1. Under Settings > General take note of the API key, it will be needed later.

1. Initial Prowlarr setup

   1. Navigate to https://prowlarr.mydomain.com
   1. Click on indexers and add whatever indexers you have.
   1. Navigate to Settings > General and click on Show Advanced.
   1. Change application URL to http://prowlarr:9696
   1. Navigate to Settings > Apps and add Radarr. Use the following settings:
      - Prowlarr Server: http://prowlarr:9696
      - Radarr Server: http://radarr:7878
      - API Key: Use the radarr API key you copied in previous step
   1. Now add the Sonarr app. Use the following settings:
      - Prowlarr Server: http://prowlarr:9696
      - Sonarr Server: http://sonarr:8989
      - API Key: Use the sonarr API key you copied in previous step
   1. Navigate to Settings > Download Clients and add your NZBGet download client. Use the same settings that you used when configuring the download client in Sonarr and Radarr.

1. Set up quality profiles

   - This is a very subjective topic, but there are three common options I've found:
     - Manually updating radarr/sonarr
     - [Following trash-guides recommendations](https://trash-guides.info/Radarr/radarr-setup-quality-profiles/), either manually or via Recyclarr
     - Dictionarry/Profilarr
   - I chose dictionarry/profilarr because I liked that you could track changes via git, and having a single UI to update and sync to both radarr and sonarr. Setting up those quality profiles is outside the scope of this guide because it's so subjective, but these documentation pages helped a lot:
     - [Profilarr setup](https://dictionarry.dev/wiki/profilarr-setup)
     - [Profile builder](https://dictionarry.dev/builder)
     - [Quality profiles reference](https://dictionarry.dev/profiles)

1. Set up Pullio to update your Docker containers
   - Run the following command to install the script on your server:
   ```bash
   sudo curl -fsSL "https://raw.githubusercontent.com/hotio/pullio/master/pullio.sh" -o /usr/local/bin/pullio
   sudo chmod +x /usr/local/bin/pullio
   ```
   - Create a crontab with
   ```bash
   sudo crontab -e
   ```
   - Paste the contents of crontab.txt from this repo into the file and save it. This crontab runs the script once a day. For different schedules use a different [cron syntax](https://crontab.guru/)
1. OPTIONAL: Set up prowlarr notifications
   1. Navigate to https://prowlarr.mydomain.com
   1. Go to Settings > Notifications
   1. Add a Discord connection, copy the webhook you created from Discord and select the notifications you want.
1. Access your homepage dashboard at https://mydomain.com. You should see links for all your applications and basic disk/RAM info.

## Misc

### [Helpful Docker Compose commands](https://trash-guides.info/File-and-Folder-Structure/How-to-set-up/Docker/#docker-compose-commands)

- sudo docker-compose up -d (This Docker-compose command helps builds the image, then creates and starts Docker containers. The containers are from the services specified in the compose file. If the containers are already running and you run docker-compose up, it recreates the container.)
- sudo docker-compose pull (Pulls an image associated with a service defined in a docker-compose.yml)
- sudo docker-compose down (The Docker-compose down command also stops Docker containers like the stop command does. But it goes the extra mile. Docker-compose down, doesn’t just stop the containers, it also removes them.)
- sudo docker system prune -a --volumes --force (Remove all unused containers, networks, images (both dangling and unreferenced), and optionally, volumes.)

## Credits

- [TRaSH-Guides](https://trash-guides.info/)
- [Another sample docker compose](https://github.com/AdrienPoupa/docker-compose-nas)
  - This one has Jellyfin, qBittorrent, and PIA VPN
