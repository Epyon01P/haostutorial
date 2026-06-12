# haostutorial: remote access

[![CC BY 4.0][cc-by-shield]][cc-by]

[![Home Assistant official logo](https://github.com/home-assistant/home-assistant.io/blob/current/source/images/blog/2023-09-ha10/home-assistant-logo-new.png?raw=true)](https://www.home-assistant.io/)

How to make a Home Assistant OS installation securely accessible from the internet using DuckDNS, Let's Encrypt and Nginx, plus a few other apps and tweaks that make every Home Assistant better.

## About

A while ago I wrote [a tutorial](https://github.com/Epyon01P/hasstutorial) on how to install Home Assistant as a stand-alone Python application (Home Assistant Core) on a regular Linux distribution like Raspberry Pi OS. It's a great way to learn some Linux along the way, and I still stand by it if you want to squeeze multiple services onto a single Pi.

However, for the vast majority of people, I no longer recommend that route. **Home Assistant OS (HAOS) is the way to go now.** It's easier to install, it manages itself, it gives you painless backups, and crucially it unlocks the whole world of *apps* (the click-to-install applications formerly known as *add-ons*) and *HACS*, the Home Assistant Community Store for custom integrations. You lose almost nothing and gain a lot of convenience.

So this is the spiritual successor to that first tutorial. It assumes you already have a working Home Assistant OS install that you can reach on your local network, and it picks up from there. We will make your Home Assistant securely reachable from anywhere on the internet, **without paying for a subscription and without relying on any third-party tunnel**, using only free apps that run on your own hardware. Along the way we'll also install a handful of apps I think every Home Assistant deserves.

And, as in the previous tutorial, I'll take the time to explain *how* the technologies we use actually work. The goal isn't not only to get you remote access; it's also for you to understand what a reverse proxy is, what a TLS certificate proves, and why MQTT is so popular, so you can troubleshoot and build on this yourself.

This guide was written against Home Assistant **2026.6.2**. 

## Table of contents

[Why expose Home Assistant this way?](#why-expose-home-assistant-this-way)
- [How does this compare to Tailscale, WireGuard and Cloudflare Tunnel?](#how-does-this-compare-to-tailscale-wireguard-and-cloudflare-tunnel)
- [A fair word on security](#a-fair-word-on-security)

[Before you begin](#before-you-begin)

[Getting a DuckDNS domain name](#getting-a-duckdns-domain-name)

[Forwarding ports on your router](#forwarding-ports-on-your-router)

[Installing the File Editor app](#installing-the-file-editor-app)
- [Trusting the reverse proxy](#trusting-the-reverse-proxy)

[Keeping your domain pointed to your external IP with DuckDNS and securing it with Let's Encrypt](#keeping-your-domain-pointed-to-your-external-ip-with-duckdns-and-securing-it-with-lets-encrypt)

[Setting up the Nginx reverse proxy](#setting-up-the-nginx-reverse-proxy)

[Accessing Home Assistant remotely](#accessing-home-assistant-remotely)

[Giving someone temporary access: a second user](#giving-someone-temporary-access-a-second-user)

[More useful apps](#more-useful-apps)
- [MQTT: the Mosquitto broker](#mqtt-the-mosquitto-broker)
- [HACS: the Home Assistant Community Store](#hacs-the-home-assistant-community-store)
- [Installing a custom integration through HACS](#installing-a-custom-integration-through-hacs)

## Why expose Home Assistant this way?

You can already control your Home Assistant from anywhere in your house. But the smart home really comes into its own when you can also flick a light, check a camera or unlock a door while you're sitting on a beach two countries over. That requires your Home Assistant to be reachable from the internet.

The easiest option, by far, is to **pay for [Home Assistant Cloud (Nabu Casa)](https://www.nabucasa.com/)**, a subscription that handles remote access (and cloud-based voice assistants) for you. It directly funds Home Assistant development, and there is absolutely nothing wrong with choosing it. If you'd rather not tinker, stop reading and subscribe.

But if you'd like to do it yourself for free, there are several popular approaches, and it's worth understanding how they differ before you pick one. They split into two philosophies: methods that keep Home Assistant **private** and only accessible through an encrypted tunnel, and methods (like this guide) that **publish** Home Assistant at a real, public HTTPS address.

### How does this compare to Tailscale, WireGuard and Cloudflare Tunnel?

I'm not an expert on every one of these, and they're all perfectly good. But here's my honest read on the trade-offs, particularly around how much fuss they put on the *client* side, meaning every phone, tablet or laptop you actually want to use:

- **[WireGuard](https://www.wireguard.com/)** is the purist's choice: a fast, modern, self-hosted VPN, with an official Home Assistant app. Nothing of yours touches a third party. The cost is setup effort on *both* ends: you still have to forward a (UDP) port on your router, and crucially, every client device needs the WireGuard app installed plus its own little key/config file (usually scanned in as a QR code). It's the most manual option to live with day to day.
- **[Tailscale](https://tailscale.com/)** is a mesh VPN built on top of WireGuard that takes most of that pain away (again, with an official Home Assistant app, and no port forwarding needed). It's lovely, with one catch: every device you want to connect from still needs the Tailscale app installed and signed in, and the connection setup is coordinated through Tailscale's own servers (a third party), even though your actual traffic stays encrypted end to end. Great when it's just you and a couple of devices; awkward the moment you want to hand a dashboard link to a houseguest or a family member who isn't going to install a VPN.
- **[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/)** is the interesting one, because on the convenience axis it's actually similar to this guide: an outbound-only tunnel means no port forwarding and your home IP stays hidden, and you reach Home Assistant at a normal HTTPS URL with no client software to install. The catches are that *all* your traffic flows through Cloudflare, who terminate the encryption and could in principle inspect it (a third party sitting in the middle of your home automation), you need a domain managed by Cloudflare, and there's a long-running grey area about whether pushing video, such as remote camera streams, through the free plan runs afoul of their terms. For a camera-heavy Home Assistant, that ambiguity is a real consideration.

This guide covers the remaining option: **roll your own public access** with DuckDNS, Let's Encrypt and Nginx. Its standout advantage is that there is *zero* client-side anything. Any web browser, any phone, the Home Assistant companion app, they all just open a plain `https://you.duckdns.org` address. Nothing to install, nothing to configure per device, and trivial to share with family. It's fully self-hosted with no third party in the data path, and the connection is properly encrypted with a certificate that every browser already trusts. For a lot of people, that "it just works from any device" quality is worth more than anything else.

At a glance:

| Method | Client setup (per device) | Port forwarding | Third party in data path | Login page on public internet | Cost |
|---|---|---|---|---|---|
| **Nabu Casa** | None | No | Yes (Nabu Casa) | Yes (via cloud URL) | ~€7.50/month |
| **WireGuard** | App + per-device config | Yes (UDP) | No | No | Free |
| **Tailscale** | App + sign-in | No | Coordination only\* | No | Free (personal use) |
| **Cloudflare Tunnel** | None | No | Yes (Cloudflare) | Yes (IP hidden) | Free |
| **DuckDNS + Let's Encrypt + Nginx** (this guide) | None | Yes (TCP 443) | No | Yes | Free |

\* With Tailscale your actual traffic is encrypted end to end and normally flows device-to-device; only the key exchange and connection setup (plus an occasional relay fallback) pass through Tailscale's servers.

No option is "best" in the abstract: a VPN keeps your login page hidden but asks every device to run software, while this guide and Cloudflare Tunnel trade a publicly reachable login page for the convenience of working from anything. Pick the row whose trade-offs you're happiest with.

### A fair word on security

The one genuine cost of all that convenience is this: unlike the VPN and tunnel methods, which keep Home Assistant *invisible* to anyone who isn't already authenticated, this approach **puts your Home Assistant login page on the open internet**. Anyone who knows or guesses your domain can at least reach that login screen.

Is that dangerous? In practice, no, probably thousands of people run Home Assistant exactly this way, the Let's Encrypt + Nginx layer is genuinely solid, and the encryption is real. However, there are some attention points (which are, to be honest, very straightforward):

- **Keep Home Assistant updated.** Updates are how any vulnerabilities that get detected are patched. On HAOS this is a couple of clicks, so there's no excuse.
- **Use a long, unique password** for every Home Assistant account.
- **Optionally: enable multi-factor authentication (MFA/TOTP)** under your Home Assistant user profile. For a publicly reachable login page, this is the single biggest improvement you can make.

## Before you begin

This tutorial assumes the following:

- You have a working **Home Assistant OS** installation, version 2026.2 or newer (we used 2026.6.2). If you don't, follow the [official installation guide](https://www.home-assistant.io/installation/) first, get through onboarding, and come back here. The rest of this guide lives entirely in the Home Assistant web interface, so it doesn't matter whether HAOS runs on a Raspberry Pi, a mini-PC, or a virtual machine.
- You can reach Home Assistant on your local network, typically at something like `http://homeassistant.local:8123` or `http://192.168.1.50:8123`. **Write down that local IP address**, you'll need it more than once.
- You have an *administrator* account in Home Assistant (the apps and settings we use require it).
- You have access to your internet router's settings, because we'll need to forward a couple of ports.

> A note on terminology: in the 2026.2 release Home Assistant renamed *add-ons* to *apps*, and moved the panel into the main interface. So when older guides or forum posts say "install the add-on", they mean exactly what this guide calls "install the app". They're the same thing. You'll find everything under `Settings` > `Apps`.

## Getting a DuckDNS domain name

To reach your home from the internet, you need to connect to your router's *public* IP address, the address your ISP assigned you. There are two problems with this. First, it's an ugly string of numbers like `78.125.36.15` that nobody wants to memorise. Second, and worse, most ISPs hand out **dynamic** IP addresses, meaning yours can silently change at any moment, breaking your connection.

A **dynamic DNS** (DynDNS or DDNS) service solves both. It gives you a friendly, permanent domain name, and it keeps that name pointed at your router's current public IP, automatically updating whenever the IP changes. Type the name into a browser anywhere in the world and you'll land at your house.

We'll use **[DuckDNS](https://www.duckdns.org/)**, a free dynamic DNS service. It lets you create up to five subdomains of `duckdns.org`, and conveniently it integrates with Let's Encrypt, which we'll lean on later.

Head to [duckdns.org](https://www.duckdns.org/) and log in. You can sign in with several providers. Google single sign-on is probably the quickest, but I'd recommend going through the small extra hassle of creating a [GitHub](https://github.com/) account and logging in with that instead, because **we'll need a GitHub account later for HACS** anyway. Kill two birds with one stone.

Once you're logged in, the DuckDNS main page shows your personal **token** near the top. Treat this like a password and keep it private, it's what authorises updates to your domains.

Below the token, you can add a domain. Pick something short and memorable, this becomes your public address. For this tutorial I'll use `hasstutorial.duckdns.org`. **Everywhere you see `hasstutorial` from here on, substitute your own chosen name.**

When you add the domain, DuckDNS automatically fills in your router's current public IP. Good. We'll teach the DuckDNS app to keep that up to date in a later step.

## Forwarding ports on your router

Your router is a gatekeeper. By default it blocks all incoming connections from the internet that weren't requested by a device inside your network. That's a sensible security default, but it also means the outside world currently can't reach your Home Assistant at all.

To let specific traffic through, we use **port forwarding**: we tell the router that when traffic arrives on a given port from the internet, it should be forwarded to a specific device and port on the local network, instead of being dropped.

You need to forward two ports to the local IP address of your Home Assistant:

- Forward external port `80` → your Home Assistant IP, port `8123` (plain HTTP, used during initial setup and for the certificate challenge).
- Forward external port `443` → your Home Assistant IP, port `443` (encrypted HTTPS, which our Nginx proxy will serve).

How you do this depends entirely on your router brand and ISP, so you're somewhat on your own here. Port forwarding is sometimes hidden under names like *NAT forwarding*, *Port mapping* or *Virtual server*. Good luck.

> **As a concrete example, here's how it looks on a UniFi router:**
> Go to `Settings`, then `Port Forwarding`, then `Create New`. Give the rule a name, e.g. `Home Assistant HTTP`. Set `WAN Port` to `80`, set `Forward IP Address` to your Home Assistant's local IP, and set `Forward Port` to `8123`. Save it. Then create a second rule, e.g. `Home Assistant HTTPS`, this time using `443` for **both** the `WAN Port` and the `Forward Port`.

We'll tighten this up later: once HTTPS is working, the port `80` rule is no longer needed and we'll remove it. But for now, set up both.

## Installing the File Editor app

Our first app does two things at once: it gives us a handy in-browser text editor, and it lets us make the one configuration change Home Assistant needs in order to trust our upcoming reverse proxy.

The **File Editor** app is a simple, browser-based editor for the files that live inside your Home Assistant configuration folder. On Home Assistant OS you don't have shell access by default, so this app is the friendliest way to edit `configuration.yaml`, the central file where Home Assistant keeps settings that aren't (yet) exposed in the UI. It's genuinely useful well beyond this tutorial, so it's worth keeping around.

Let's install it:

1. In Home Assistant, go to `Settings` > `Apps`.
2. In the bottom-right corner, click `Install app`.
3. In the *Official apps* list, search for **File editor** (its subtitle reads *Simple browser-based file editor for Home Assistant*). Select it and click `Install`.
4. Once it has installed, click `Start`.

### Trusting the reverse proxy

Now we'll use File Editor to make one small but essential change.

Here's the why. When Nginx, the reverse proxy we will install in the next steps, forwards a request from the internet to Home Assistant, the request arrives at Home Assistant looking like it came from Nginx, not from the original visitor. By default Home Assistant is suspicious of this: it doesn't know whether to trust whatever is in front of it, and it will reject proxied requests outright. We need to explicitly tell Home Assistant, "relax, the thing forwarding these requests is *my own* reverse proxy, you can trust it." That's what the `trusted_proxies` setting does.

On Home Assistant OS, apps run in isolated containers connected by an internal Docker network. The Nginx app will reach Home Assistant from an address inside the `172.30.33.0/24` range, so that's the range we'll mark as trusted.

To make the change:

1. Go to `Settings` > `Apps` > `File editor`.
2. Click `Open web UI`.
3. In the top-left corner, click the folder icon (`Browse Filesystem`).
4. In the file list, select `configuration.yaml`. Its contents appear in the right-hand pane, where you can edit them.
5. Just below the existing `default_config:` line, leave a blank line and then add the following block. Indentation matters in YAML, so copy it exactly (two spaces, then four spaces, as shown):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.30.33.0/24
```

6. Make sure there's also a blank line *after* this block, separating it from whatever follows. YAML is fussy about its blank lines and indentation.
7. In the top-right corner, click the `Save` icon (the diskette).

Finally, Home Assistant needs to reload its configuration for the change to take effect. Go to `Settings` > `System`, click the power button in the top-right corner, and select `Restart Home Assistant`. Confirm, then wait a couple of minutes for Home Assistant to come back.

## Keeping your domain pointed to your external IP with DuckDNS and securing it with Let's Encrypt

We now have a domain name and a way to reach our network. Next we need **encryption**, so that everything travelling between you and your home over the internet is unreadable to anyone in between.

Encrypted web traffic (HTTPS) relies on a **TLS certificate**. A certificate does two jobs. It carries the cryptographic key used to encrypt the connection, and it *proves* that the server you're connecting to really is the one that owns your domain, and not some impostor who hijacked your address. That proof comes from a **Certificate Authority (CA)**, a trusted third party that verifies ownership and issues the certificate as evidence. Your browser ships with a list of CAs it trusts; if a site presents a certificate signed by one of them, the browser shows a reassuring padlock. If not, it throws up a big scary warning.

Getting such a certificate used to be slow and expensive. I still remember having to fax a signed company letterhead to a CA to validate a website back in 2012. Those days are gone, thanks to **[Let's Encrypt](https://letsencrypt.org/)**, a non-profit CA that issues certificates automatically and for free. The certificates are short-lived (they expire after a few months, which is actually much safer than long-living certificates), but the whole issue-and-renew cycle is automated, so you never have to think about it.

The DuckDNS app conveniently bundles both jobs: it keeps your domain pointed at your changing IP address, *and* it requests and renews a Let's Encrypt certificate for that domain.

1. Go to `Settings` > `Apps`, click `Install app`, and search the *Official apps* list for the **Duck DNS** app (subtitle: *Free Dynamic DNS (DynDNS or DDNS) service with Let's Encrypt support*). Install it.
2. Open the app and click `Configuration` in the ribbon at the top.
3. You'll see an `Options` panel. **If it already contains an empty grey field with just an `X` under *Domains*, delete it first** by clicking the `X`. (A leftover empty entry will cause the app to fail.)
4. Click the `Domains` button (the one with a `+` in front of it), type your full domain (here, `hasstutorial.duckdns.org`), and click `Add custom item`.
5. In the `Token` field, paste the DuckDNS token from the DuckDNS website.
6. Expand the `Lets Encrypt` section and toggle `accept_terms` to *on* (you're agreeing to Let's Encrypt's terms of service).
7. Click `Save`.

Now switch to the `Info` tab and click `Start`. If you're curious, open the `Log` tab to watch it work: the app will register your domain and then run the Let's Encrypt certificate signing process. After a short while you should see a triumphant `Done!` line, meaning your certificate has been issued and stored.

> **You can now remove the router rule that forwards external port `80` to `8123`.** That rule was for unencrypted HTTP traffic. Now that we have a signed TLS certificate, all our remote traffic will go through encrypted HTTPS on port `443`, which our Nginx proxy is about to handle. Keeping port `80` open to plain Home Assistant serves no further purpose, so close it. (Leave the `443` → `443` rule in place.)

## Setting up the Nginx reverse proxy

This is the keystone of the whole setup. We have a domain, and we have a certificate. Now we need something to actually *receive* the encrypted connections from the internet, decrypt them, and hand the requests over to Home Assistant. That something is **Nginx** (pronounced *Engine X*), and we'll use the purpose-built **NGINX Home Assistant SSL proxy** app.

Why bother with a proxy at all, instead of just exposing Home Assistant directly? Picture all the services running on your Home Assistant OS you might one day want to reach from outside as houses on a street. If every house has its own door opening straight onto the public road, each one is a separate thing for a burglar to rattle. Not very safe.

A reverse proxy rearranges those houses around a private courtyard with a single guarded entrance. Nginx is the doorman at that entrance. Every visitor arriving from the internet has to go through Nginx first. Nginx checks which domain they asked for, and if it recognises it, politely escorts the request to the correct service inside, here, Home Assistant on port `8123`. Anyone showing up without a valid request gets turned away at the gate.

On top of that, Nginx handles all the encryption. Home Assistant keeps speaking simple, fast, unencrypted HTTP on your local network, while Nginx wraps everything in HTTPS the moment it leaves your house. The encryption lives entirely in one place, and Home Assistant never has to worry about it.

Let's set it up:

1. Go to `Settings` > `Apps`, click `Install app`, and search the *Official apps* list for the **NGINX Home Assistant SSL proxy** app (subtitle: *An SSL/TLS proxy.*). Install it.

> Be careful not to confuse this with the similar-sounding **Nginx Proxy Manager** app, which is a different, broader tool. For this tutorial we want the *NGINX Home Assistant SSL proxy*.

2. Once installed, go to the `Configuration` tab in the top ribbon.
3. In the `Domain` field, fill in your DuckDNS domain (e.g. `hasstutorial.duckdns.org`). **Leave everything else exactly as it is.** The defaults already point the proxy at the certificate the DuckDNS app generated.
4. Click `Save`.
5. Switch to the `Info` tab and click `Start`.

Open the `Log` tab to watch the magic. Nginx now sets up the security layer for your Home Assistant, including generating a DSA key, which on a modest device like a Raspberry Pi can take a little while. When it's finished, the log will settle on a line reading `Running nginx...`.

That's it. Your reverse proxy is live.

## Accessing Home Assistant remotely

The moment of truth. Open a browser and go to your domain over HTTPS:

```
https://hasstutorial.duckdns.org
```

(substituting your own domain). You should be greeted by the familiar Home Assistant login screen, served over an encrypted connection with a valid certificate, look for the padlock in the address bar. Notice that you don't have to add `:8123` to the URL: Nginx is listening on the standard HTTPS port `443` and quietly forwarding to Home Assistant's `8123` behind the scenes.

To really prove it works, test from *outside* your home network. The cleanest way is to grab your phone, **turn off Wi-Fi** so it uses mobile data, and open your domain in the phone's browser. If the login page appears, congratulations, your Home Assistant is now securely reachable from anywhere on the planet, hosted entirely on your own hardware, with no subscription and no third-party tunnel in the middle.

A couple of finishing touches worth doing:

- In Home Assistant, go to `Settings` > `System` > `Network` and set your **external URL** to `https://hasstutorial.duckdns.org`. This helps various integrations and notifications generate correct links.
- In the **Home Assistant companion app** on your phone, set the external URL to the same address. If you also tell the app your home Wi-Fi network name, it will automatically use the fast local connection when you're home and switch to the DuckDNS address when you're away, giving you the best of both worlds.

> Troubleshooting: if the page won't load remotely, the usual suspects are (in order) the router port-forward for `443`, a typo in the domain field of the Nginx app, or the Nginx app not having finished starting. Check the Nginx app's `Log` tab first. If you see a *Bad Request* or a blank page, double-check that the `http:` block you added to `configuration.yaml` is correct and that you restarted Home Assistant after adding it. A surprising number of issues come from Home Assistant rejecting the proxy because `trusted_proxies` is missing or wrong.

## Giving someone temporary access: a second user

Now that your Home Assistant is reachable from the internet, you can let someone you trust log in remotely to help you out, and then cut that access off again when they're done. This is, in fact, one of the reasons I wrote this guide: a friend can set up remote access using these steps, hand me a temporary account to configure something tricky, and disable it the moment I'm finished.

Home Assistant supports multiple user accounts, each with its own login. Creating one for a helper takes a minute:

1. Go to `Settings` > `People`, then open the `Users` tab at the top.
2. In the bottom-right corner, click `Add user`.
3. Fill in a `Display name` and a `Username`, and set a strong, unique `Password`.
4. If the person needs to change settings, install apps or edit configuration (i.e. actually set things up for you), toggle `Administrator` **on**. For a guest who only needs to view dashboards, leave it off.
5. Make sure `Can only log in from the local network` is **off**, otherwise the account won't work over your DuckDNS address.
6. Click `Create`.

Now share the username and password with your helper through a secure channel (a password manager's share feature, or at least not plain email), along with your `https://yourdomain.duckdns.org` address. They can log in from anywhere and get to work.

> A word of caution: an administrator account can do *anything* on your Home Assistant, so only hand one out to someone you genuinely trust, and revoke it promptly when the work is done.

When you no longer need the account, you have two options. To keep it around for next time but block all access, open the user in `Settings` > `People` > `Users` and toggle `Active` **off**, which instantly disables logins without deleting anything. To remove it entirely, open the user and click `Delete user`. Either way, the temporary door is closed again.

## More useful apps

Remote access was the main event, but while we're here, let me point you at three things that genuinely improve almost any Home Assistant. The first is a messaging backbone for your devices, and the other two unlock a whole ecosystem of community-made integrations.

### MQTT: the Mosquitto broker

If there's one foundational tool that benefits nearly every Home Assistant install, it's an MQTT broker.

**MQTT** (Message Queuing Telemetry Transport) is a lightweight publish-subscribe messaging protocol. It's enormously popular in the home automation world because it's simple yet flexible and barely uses any resources, which matters when your "computers" are tiny battery-powered sensors.

Picture an MQTT **broker** as a wall of mailboxes, where each mailbox is called a **topic**. A client, say, a little IoT temperature sensor, can **publish** a **message** (the latest temperature reading) into a topic. Another client, like Home Assistant, **subscribes** to that topic and gets notified the instant a new message arrives. The beauty is that publishers and subscribers don't need to know anything about each other; they only need to agree on a topic name. Any client can invent any topic on the fly, and the broker creates it as needed. If a message is published with the **retain** flag, the broker keeps the last value on hand so a newly-connecting subscriber immediately sees the current state. All of this happens in near real-time with minimal overhead, which is exactly what you want for a smart home.

Home Assistant makes this painless. You don't even have to install the broker app by hand, the integration sets it up for you:

1. Go to `Settings` > `Devices & services`.
2. In the bottom-right corner, click `Add integration`.
3. Search for `MQTT` and select it. On the next screen, select `MQTT` again.
4. When prompted, choose `Use the official Mosquitto MQTT Broker App`. Home Assistant will install and wire up the broker automatically.

Wait a moment, and you're done, Home Assistant can now send and receive MQTT messages. From now on, any MQTT-capable device on your network that supports Home Assistant's discovery will simply show up under the MQTT integration, ready to use.

> Good to know: your broker's address is now your Home Assistant's local IP, on port `1883`. You'll need those details when configuring other devices or services (for example ESPHome nodes, Zigbee2MQTT, or a Frigate NVR) to talk to your broker.

### HACS: the Home Assistant Community Store

Home Assistant ships with hundreds of official integrations, but getting a brand-new integration accepted into the core project is a long road. In the meantime, the community publishes an enormous amount of excellent **custom** integrations on GitHub.

**HACS** (the Home Assistant Community Store) is the tool that makes installing and updating these community integrations a matter of a few clicks, instead of manually copying files around. It's arguably the single most valuable thing you can add to Home Assistant OS, and one of the biggest reasons HAOS beats the manual Core install for most people. (Back on Home Assistant Core, you had to clone Git repositories by hand for every custom integration. HACS does all that for you.)

The official HACS download instructions live at [hacs.xyz](https://www.hacs.xyz/docs/use/download/download/), but since they involve a couple of slightly unintuitive steps, we'll walk through it here.

First, add the HACS app repository to your app store so Home Assistant can find the installer:

1. Go to `Settings` > `Apps`, then click `Install app`.
2. In the App store overview, click the three dots in the top-right corner and select `Repositories`.
3. In the dialog, in the bottom-right corner, click `+ Add`, paste `https://github.com/hacs/addons` as the URL, and click `Add`.

Now install the HACS bootstrapper app:

4. Still in `Settings` > `Apps`, click `Install app`.
5. Scroll down (or use the search box) and select `Get HACS`. Click `Install`.
6. When it has installed, click `Start`, then open the `Log` tab and watch the output scroll by. Once it reaches a line about having *successfully stopped*, the app has done its job.
7. Restart Home Assistant: `Settings` > `System`, power button in the top-right, `Restart Home Assistant`.

After Home Assistant restarts, add HACS as an integration:

8. Go to `Settings` > `Devices & services`, click `Add integration`, and search for `HACS`.

> If `HACS` doesn't appear in the list, your browser is probably showing a cached page. Do a hard refresh (`Shift` + `F5`) or clear the cache, and try again.

9. When prompted, check **all** the boxes and click `Submit`.

This is where that GitHub account pays off. HACS authenticates against GitHub, so if you don't have an account yet, [now's the time to make one](https://github.com/signup). HACS will show you a code and a link; click the link, paste the code on GitHub, and click `Authorize hacs`. Back in Home Assistant, click `Skip and finish`.

You should now have a `HACS` entry in the Home Assistant sidebar. (If it's missing, hard-refresh the browser once more.)

### Installing a custom integration through HACS

Let's put HACS to work with a real example. We'll install my own [**Peak Power Forecast**](https://github.com/plan-d-io/Peak-Power-Forecast) integration.

A quick word on what it does, in case it's useful to you. In Flanders (and increasingly elsewhere in Belgium), part of your electricity bill is the so-called **capacity tariff** (*capaciteitstarief*): you're charged based on your highest 15-minute *average* power draw in a given month. Switch on the oven, the dishwasher, the EV charger and the kettle all at once, and that single expensive quarter-hour can set your network cost for the whole month. Peak Power Forecast watches your live consumption and forecasts where your monthly peak is heading, so you can spot a peak forming and shift some load before it costs you. If you're on a capacity tariff, it's a handy thing to have on a dashboard.

Whether or not that integration is relevant to you, the workflow below is identical for **any** custom integration distributed through GitHub, so it's worth knowing:

1. Open the `HACS` tab in the sidebar.
2. In the top-right corner, click the three dots and select `Custom repositories`.
3. Paste `https://github.com/plan-d-io/Peak-Power-Forecast` into the `Repository` field, set `Type` to `Integration`, and click `Add`.

> A small HACS quirk: after you click `Add`, the dialog stays open. It *has* added the repository, you can tell because you could now remove it again via the red bin icon, it just doesn't close itself or give you a clear "done". Just click the `X` to dismiss the dialog.

4. In the search field, type `Peak Power Forecast` and select it.
5. In the bottom-right corner, click `Download` and follow the prompts.

And that's how you install custom integrations through HACS. After downloading, you'll usually be asked to restart Home Assistant, after which the new integration becomes available under `Settings` > `Devices & services` > `Add integration`, just like any built-in one.

From here, the whole HACS ecosystem is open to you. Explore, experiment, and enjoy your now globally-reachable, self-hosted smart home.

## License

This work is licensed under a
[Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
