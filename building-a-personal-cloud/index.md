# Building a Personal Cloud


Like many others, I have numerous services which I use frequently for work, fun, productivity, and more. While using third party services is the easiest and fastest way to fulfill a need, it is far from the most privacy friendly, or depending on your view the most secure. I decided that I wanted to change that, and in 2019 I began to replace third party hosted services with services I can host on a personal Virtual Private Server, and have complete control of the data. Below is a summary of the replacements I made:

### Hosting Locations
**Self-Hosted - Cloud**
  * Google Contacts | Nextcloud Contacts
  * Google Calendar | Nextcloud Calendar
  * Dropbox | Nextcloud / Cryptomator
  * iCloud Notes | Standardnotes
  * Text File | Nextcloud Tasks / 2do
  * Lastpass | Vaultwarden
  * Pocket | Wallabag
  * Feedly | TT-RSS
  * TeamViewer | MeshCentral
  * Nothing | Firefox Sync Server
  * Nothing | Restic Backup to Backblaze
  * Nothing | Tandoor Recipes

**Self-Hosted - Local**
  * Nothing | PiHole + Unbound + DoH
  * Nothing | PiVPN - Wireguard
  * iCloud Computer Backup | NAS Time Machine
  * Amazon Cloudcam | DaFang Hacks / VLC / Telegram

I also took this opportunity to move from Gmail to ProtonMail for email. While email can be self-hosted as well, I felt that swapping the very privacy-unfriendly Google service for a trusted encrypted solutions provider (ProtonMail) met my goal.

In 2020 I went through a migration from Traefik 1.7 to 2.4, and took the opportunity to completely revamp my self hosting approach, applications, best practices, security, and more. I'm hosting everything on a Contabo VPS and on a local Raspberry Pi.

**Local Raspberry Pi**

* [Unbound](https://nlnetlabs.nl/projects/unbound/about/) \- Local recursive DNS resolver
* [PiHole](https://pi-hole.net/) \- DNS black hole
* [Wireguard (PiVPN)](https://www.pivpn.io/) \- VPN Access through PiVPN

I bought several cheap Wyze 2.0 Cameras and put the [DaFang custom firmware](https://github.com/EliasKotlyar/Xiaomi-Dafang-Hacks) on them this lets me VPN into my home network and connect to network streams from the cameras using VLC on my phone. I have motion detection notifications running through Telegram with stills taken when motion is detected

**Contabo VPS - 9 Euro per month, 6 Cores, 16GB Ram, 400GB Storage (SSD)**

As part of my security updates, I decided that some services I host had no business being externally accessible, even when they're behind HTTP authentication. I set up wireguard docker image to allow me to access my VPS over VPN, and then modified wireguard's Coredns to allow me to access the services I want on the domains I would have previously accessed externally. I followed [this guide](https://noobgasm.com/tutorial-additional-protection-with-wireguard/) for that.

Other security updates include moving to official images for everything, leveraging Cloudflare's proxy service for any web based applications (though some may argue this isn't as good for privacy), and moving to docker secrets where possible

I also moved to a socket proxy for the docker socket rather than allowing any images (except for the socket proxy itself) direct access to the docker socket.

Everything except fail2ban on my Contabo VPS are sourced from docker images.

### Web Applications hosted behind Wireguard VPN

* [Portainer](https://www.portainer.io/) \- Docker Management
* [PhpMyAdmin](https://www.phpmyadmin.net/) \- Management of a MariaDB database for applications which don't support Postgres
* [PgAdmin](https://www.pgadmin.org/) \- Management of a Postgres database for any applications which do support Postgres
* [Redis Commander](https://github.com/joeferner/redis-commander) (Currently Disabled) \- for managing my Redis Memcache installation to speed up Nextcloud

### Non-Open-web accessible applications

* [Docker Socket Proxy](https://github.com/fluencelabs/docker-socket-proxy) \- This allows applications to access only the services they need from the socket proxy and nothing else
* [Watchtower](https://github.com/containrrr/watchtower) \- Docker image updates, used as Ouroborus is no longer actively updated
* [MariaDB](https://mariadb.org/) \- Open source MySQL server for applications which don't support my preferred SQL server, Postgres
* [Postgres](https://www.postgresql.org/) \- Open source SQL server for applications which it
* [Redis](https://redis.io/) \- Memcache to speed up Nextcloud
* [Restic/Resticker](https://github.com/djmaze/resticker) \- This is a docker image which contains a parameterized version of Restic, for automated, encrypted, incremental backups. I'm backing up to a Backblaze B2 bucket which is low cost and use based.

### Open-web accessible applications

* [Traefik 2.4](https://traefik.io/) \- used for reverse proxying open-web applications
* [Firefox Syncserver](https://github.com/mozilla-services/syncserver) (MariaDB) - this is in the process of being moved to Rust, but that image isn't ready yet. Still using the old python 2.7 version to sync browser settings, extension lists, and bookmarks.
* [Wallabag](https://www.wallabag.it/en) (Postgres) - Pocket replacement, this integrates very well with iOS and allows me to "stash" anything I come across on twitter, reddit, browsing, and more. I also have Wallabag set up as an RSS feed so when I stash something it shows up in my RSS reader.
* [Nextcloud](https://nextcloud.com) (Postgres) - primarily used as Google replacement, and I use it for Contacts, Calendar, Drive, and Tasks/ToDo hosting
* [Wireguard](https://github.com/linuxserver/docker-wireguard) \- allows me to access internal services via VPN. This is one of the few "unofficial" images I use from linuxserver.io as I don't think wireguard hosts an official one
* [Tiny-Tiny-RSS](https://tt-rss.org/) (Postgres) - Last year the developer created official docker images, so I moved from an unofficially maintained one to the official ones. This consists of a backend and frontend component, as well as an updater cron job. I much prefer this approach as the unofficial image wasn't static, and was just an older docker image that pulled the latest version of the app from Git.
* [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (Postgres) - This is the unofficial bitwarden api/backend/frontend implementation in Rust. I moved from the default sqlite backend to a postgres backend and found that it is now much speedier when updating folders
* [Standardnotes Sync Server](https://github.com/standardnotes/standalone) (MariaDB) \- This is the official backend implementation for standardnotes. I was using the Nextcloud Notes before, but wasn't happy with the mobile app experience (manual sync over webdav in the Notebooks app). The official iOS app is just as seamless as the native iPhone notes app. This backend server was recently re-written in JavaScript (previously Ruby on Rails). I am not using the recommended "script" based implementation here as I prefer more control over containers and architecture, so I've implemented the standalone containers directly in my docker-compose file.
* [Standardnotes Web App](https://github.com/standardnotes/web) \- the web frontend for standardnotes
* [Standardnotes Extensions](https://github.com/eric-pierce/standardnotes-extension-server) \- This is a docker image which contains all the open source extensions for standardnotes. It's very simple to run, and lets you use several high powered extensions all self-hosted. I forked this and added additional themes and services.
* [MeshCentral](https://github.com/Ylianst/MeshCentral) \- This is a complete replacement for TeamViewer, and enables remote access to my devices without needing to use a third party service. The only downside I've found here is that there isn't a native mobile app.
* [Tandoor Recipes](https://github.com/vabene1111/recipes) \- App for self-hosting and sharing recipes. With family members spread out this is a great way to connect around favorite recipes and meal planning.

### Possible Future Services

* Bookstack
* Papermerge
* Monica CRM
* fail2ban - may move it into a container
