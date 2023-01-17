---
title: "Post-Mortem: Images/Media Unavailable"
description: Postmortem analysis of a partial media outage on January 16, 2023.
author: tswan
permalink: /jan-16-2023-images-media-unavailable/
tag: incident
layout: post
---

On Monday, January 16, 2023, at roughly 6:00pm ET, we received our first report of images not loading on the nutmeg.social website, as well as mobile apps. This partial outage affected a subset of users, who could not load or view user-submitted images, avatars, and other media. Difficulty reproducing the issue caused delays in isolating the root cause of the issue. Ultimately, service was restored to users at 7:00pm ET.

# Timeline

- 18:00 First report of service disruption received.
- 18:08 Second user confirms they are having issues.
- 18:10 `@tswan` begins investigating the issue, but is unable to reproduce it locally.
- 18:12 `@tswan` reviews system health, identifying a small spike in disk I/O. Some time is spent reviewing this before determining I/O latency is acceptable.
![Chart titled "Disk IOps Completed" showing a spike in both read and write activity around the time of the outage. ](/assets/images/jan-16-2023-disk-io.png)
- 18:17 `@tswan`, unable to reproduce the issue locally, [posts](https://nutmeg.social/@tswan/109701432839341851) asking for user feedback or links to specific broken images.
- 18:20 `@tswan` checks Wasabi, our S3 object storage provider, for any incidents or service disruptions–there are none.
- 18:25 `@tswan` checks OVH, our server host/provider, for any incidents or service disruptions–there are none.
- 18:36 Users confirm they are still having problems.
- 18:38 `@tswan` reviews firewall rules to see if users are being inadvertently banned or rate-limited.
- 18:42 `@tswan` checks `nutmeg.social` from a cellphone on the AT&T network and does not have issues loading images.
- 18:44 `@tswan` checks `nutmeg.social` from a cellphone on the Verizon network and is finally able to reproduce the issue.
- 18:48 `@tswan` tethers a laptop to the AT&T phone to perform a traceroute to `media.nutmeg.social`, seeing no abnormalities.
- 18:51 `@tswan` tethers a laptop to the Verizon phone to perform a traceroute to `media.nutmeg.social`, and discovers that it is connecting via IPv6.
- 18:55 `@tswan` reviews Nginx configuration, identifies a missing setting, and reloads the webserver.
- 19:00 Users confirm that service is restored.

# Root Cause

The cause of the partial outage was a missing `listen` statement in our Nginx configuration.

Earlier in the day, a networking issue on our server was corrected, allowing it to connect to the Internet via IPV6 (see: [What is IPv6?](https://www.geeksforgeeks.org/what-is-ipv6/)). When connectivity was established, DNS records for both `nutmeg.social` and `media.nutmeg.social` were created to route IPv6 connectivity to our server.

For nutmeg.social, our Nginx (webserver) configuration included both a `listen 443` (IPv4) statement and a `listen [::]:443` (IPv6) statement. However, media.nutmeg.social only contained `listen 443`. As a result, IPv6 connections to our media server were rejected. Adding in the `listen [::]:443` statement allowed these connections to establish successfully.

## Things that went well

- Our users were communicative and provided screenshots demonstrating what was happening, which helped isolate the issue.

## Things that went poorly

- We did not have alerting to detect this issue.
- Difficulty reproducing the issue (`@tswan`'s internet service is IPv4-only) caused significant delays in troubleshooting and restoration.

## Opportunities

- Set up alerting for both IPv4 and IPv6 endpoints on our [status page](https://status.nutmeg.social)
