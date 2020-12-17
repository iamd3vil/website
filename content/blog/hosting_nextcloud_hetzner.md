+++
title = "My Nextcloud setup"
description = "Hosting Nextcloud for my family as an alternative to proprietary cloud."
date = 2020-12-04
slug = "hosting_nextcloud"
draft = false

[taxonomies]
tags = ["Self Hosting"]
+++

Me and my family uploads most of our photos on Google Photos. This always bothered me since this is one of the most important data we have and we are relying on the promise of unlimited storage by Google. Even then we are losing the original quality photos and uploading lossy photos (what google calls High Quality Photos) to get the unlimited storage. I always wanted to find a good alternative for this. I investigated several alternatives for taking backup of photos from our mobiles. Although there are several cloud services which does this, I didn't want to rely on another cloud service and wanted to self host this. I finally settled on `Syncthing` for myself.

I run Syncthing on one of my Raspberry Pi 4 and also run it on my mobile. It syncs all my photos from my mobile to the Raspberry Pi which kind of acts like a NAS. This works well for me but I can't do this for all my family member who are not tech savvy. So I wanted a simpler solution which can be used by anyone. Enter [Nextcloud](https://nextcloud.com). I ran it on my Raspberry Pi NAS and it ran well. I tested it with their mobile app and found that this is pretty good for our use case. I was worried that this isn't a reliable setup since it's running on a Pi at my home which has a flaky internet setup. I wanted to move this to a more reliable setup but since I was always lazy, I never did.

Recently google announced that there is [no more unlimited storage](https://blog.google/products/photos/storage-changes) in Google Photos. This motivated me to setup a more reliable Nextcloud.

### Setting up Nextcloud

There are several ways to setup Nextcloud. One setup I thought would be the best way to setup a S3 compatible object storage as the primary storage for Nextcloud on a VPS. So I have setup a Digitalocean droplet with Backblaze B2 but the problem is that the it was rather slow as the primary storage when there are lot of objects in a folder. Also I got some errors while uploading a large number of files.

I have investigated and found that the probably the most cost effective, reliable and performant way is to get a VPS with a reliable provider and use their Block storage for primary storage. I have settled on Hetzner's cloud VPS solution since this is the cheapest and they have a good reputation.

#### Setting up Hetzner VPS

I used Ansible to setup the VPS. Since there is no Cloud Firewall provided by Hetzner, we need to setup our firewall using `ufw`. 

We need to allow these ports:

- `80` (http)
- `443` (https)
- `22` (ssh)


> We are going to use Docker for setting up Nextcloud. One thing to keep in mind is that any forwarded ports will disobey `ufw` rules. This is a [known issue](https://github.com/docker/for-linux/issues/777) and there seems to be no official resolution close by. So we are only forwarding `80` & `443` and since these ports are anyways open, we are okay in this case.

A Volume of 100 GB is attached to the server while creating the VPS. We are going to use ZFS for this volume for it's compression and snapshotting capabilities.

A pool needs to be created with this volume. Let's call this `tank`.

```bash
$ zpool create tank /dev/disk/by-id/idofthedisk
```

Here instead of using something like `/dev/sdb` we using the id of the disk so that even if the disk's name changes we should be fine. You can find the id of the disk by using `ls -lah /dev/disk/by-id/`.

We will create a dataset for our Nextcloud.

```bash
$ zfs create -o compression=lz4 mountpoint=/data/nextcloud tank/nextcloud
```

This should create a dataset called `tank/nextcloud` with `lz4` compression and then mount it at `/data/nextcloud`. This will be where all our Nextcloud data will live.

For managing the containers, `docker-compose` can be used. Here is the file I used for running Nextcloud.

> Note: Replace all the variables in `{{ }}` with the correct values.

```yaml
version: '3'
services:
  nextclouddb:
    image: postgres
    restart: always
    volumes:
      - /data/nextcloud/db:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB={{ nextcloud_db_name }}
      - POSTGRES_USER={{ nextcloud_db_user }}
      - POSTGRES_PASSWORD={{ nextcloud_db_password }}
    networks:
      - "db"

  nextcloud:
    image: nextcloud:20.0.2
    restart: always
    volumes:
      - /data/nextcloud/nextcloud:/var/www/html
    environment:
      - POSTGRES_HOST=nextclouddb
      - POSTGRES_DB={{ nextcloud_db_name }}
      - POSTGRES_USER={{ nextcloud_db_user }}
      - POSTGRES_PASSWORD={{ nextcloud_db_password }}
      - NEXTCLOUD_ADMIN_PASSWORD={{ nextcloud_admin_password }}
      - NEXTCLOUD_ADMIN_USER={{ nextcloud_admin_user }}
      - NEXTCLOUD_TRUSTED_DOMAINS={{ nextcloud_domain }}
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=caddy
    depends_on:
      - "nextclouddb"
    networks:
      - "db"
      - "nextcloud"
  caddy:
    image: mrkaran/caddy:latest
    container_name: caddy
    restart: always
    volumes:
      - "/data/caddy/config:/config"
      - "/data/caddy/data:/data"
      - "/data/caddy/Caddyfile:/etc/caddy/Caddyfile"
    ports:
      - "80:80"
      - "443:443"
    networks:
      - "nextcloud"

networks:
  db:
  nextcloud:
```

Here Caddy is used as the reverse proxy to Nextcloud so that TLS certificates are managed automatically. This should make your nextcloud available at `{{ nextcloud_domain }}`.

### Backing up

I use [Restic](https://restic.net/) as my backup solution. I have a daily cron which takes backup of this to Backblaze B2.

### Expanding storage

Since this is a volume, I can always resize my volume to a larger size. I need to take my machine offline and then expand the ZFS pool once I have resized the volume. 

Another way is to add a new volume and then add that volume to the pool. There is no need to take the machine offline in this case. One thing to note here: the data isn't rebalanced between the volumes automatically, but the new data is written to the new volume until both the volumes are same in size.

### Conclusion

Nextcloud is a good alternative to all the proprietary clouds. It has a lot of quirks sure but this is being improved constantly. It isn't really not a real alternative to something like Google Photos which not only backs up the photos automatically but magically tags the photos with people & items in the photos (and other crazy ML shit Google does), but for backing up, Nextcloud __is__ a good alternative.

I plan to explore [Photoprism](https://github.com/photoprism/photoprism) as an alternative to Google Photos for their ML magic.