+++
title = "Using Mullvad VPN with Tailscale"
description = "Mullvad VPN and Tailscale running together on my mobile"
date = 2026-03-23
slug = "mullvad-tailscale"
draft = false

[taxonomies]
tags = ["networking"]
+++

I wanted to use [Mullvad VPN](https://mullvad.net/) and [Tailscale](https://tailscale.com/) together on my Android phone. The problem is Android only allows one active VPN at a time. Tailscale does have a built-in option to purchase Mullvad through them, but that's not available in India.

So I built a small tool to work around this: [mullvad-tailscale](https://git.simhadri.rocks/iamd3vil/mullvad-tailscale).

It's a Docker container that takes a Mullvad WireGuard tunnel and exposes it as a Tailscale exit node. Once it's running, it shows up as a node in your tailnet with exit node enabled. Any device on your tailnet can then route traffic through it and out via Mullvad.

It also comes with a small web UI called mullvad-switcher that lets you change the Mullvad server location on the fly.

Check out the [repo](https://git.simhadri.rocks/iamd3vil/mullvad-tailscale) for setup details.
