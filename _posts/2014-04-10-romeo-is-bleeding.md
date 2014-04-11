---
layout: default
title: Romeo is Bleeding (CVE-2014-0160)
---

This week was arguably one of the worst weeks to work
in systems operations in the history of the Internet.
The revelation of what has been called [Heartbleed](http://heartbleed.com),
a bug in OpenSSL that allows attackers to read memory
from vulnerable servers (and potentially retrieve memory from vulnerable
clients) has had many administrators scrambling. This bug makes it trivial
for hackers to obtain the private keys to a site's SSL certificate, as well
as private data that might be in-process such as usernames and passwords.

While there is a huge potential for multiple blog posts regarding our learnings
from this week, in this post I'll focus on the current state of affairs, as
well as a timeline of events.

**tl;dr** — wanelo.com **was** effected by Heartbleed. As of 1am April 8, the
public-facing parts of Wanelo were no longer vulnerable. Through the rest of
this week we have followed up to ensure that internal components are also secure.
This afternoon we deployed new SSL certificates and revoked our old ones. We
have no indication that our site was hacked, but there is no way to be certain.


*Note: Heartbleed effects OpenSSL versions 1.0.1 through 1.0.1f and 1.0.2 beta
when SSL heartbeats are enabled (the default) at compile time. On Monday, April
7 2014 version 1.0.1g was released with a fix for the bug. OpenSSL with heartbeats
disabled at compile time are not vulnerable.*

### Monday

In the mid-afternoon I was pairing with James on some
operational work relating to database upgrades when I
heard Paul mention a security vulnerability. "How bad could it be?" I wondered,
but the work we were doing had a lot of moving pieces and we were quickly lost
in the details. If you've ever been in
the zone when pairing, you might forgive me not dropping everything to check it
out.

At about 6pm, we had finished as much as we could safely deploy that day. Before
closing everything up, I decided to check up on the security problem. After finding
a writeup, I'm pretty sure my face looked a little something like this:

![Me realizing what Heartbleed meant](/assets/shelley.jpg)

By 7:30, I was home and starting to work on a fix. Up until this point we have
relied on packages to install OpenSSL, so I needed to write new automation
to compile OpenSSL from source on SmartOS. After a [very] quick check of
different sites, I did not find a Chef cookbook that already worked and chose
to write a new one. This cookbook can be found [here](https://github.com/wanelo-chef/ssl).

After successfully bootstrapping a test machine with a vulnerable version of
OpenSSL and the version of our load balancing software used in production, I
managed to compile a safe version of OpenSSL over the installed files.
By verifying that the new OpenSSL did not crash our balancer, I felt confident
that it was indeed a backwards-compatible replacement and that I could proceed.

I then quickly deployed the new OpenSSL to our demo and staging servers to test
it against our real code stack, as well as with our iOS app. Seeing that there
were no apparent problems and realizing the scope of the vulnerability, I pushed
forward and deployed OpenSSL to our production servers. In this case, the risk
of not moving forward was far greater than the risk of taking the site down. In
fact, taking the site down would have been far preferable to leaving it up in a
vulnerable state.

Happily, everything restarted correctly.

At this point I spent a little while verifying that our Ruby installations all
used dynamic linking of libssl, so that by doing a rolling restart of Unicorn
our main internal application would no longer be vulnerable. By 2am I was
reasonably confident that this was the case, understanding that other internal
applications (and our omnibus Chef installation) were still potentially at risk.
Sometime around 3am I passed out.

### Tuesday

7:30am, I awoke with an expression perhaps similar to this one:

![Was it just a nightmare?](/assets/shelley2.jpg)

After further testing of the site, I verified that our main Ruby installation
was now secure. A bit of investigation revealed that our SSL settings, however,
were not ideal. These I updated and rolled out posthaste.

Later in the day I turned my attention to other parts of our infrastructure,
ensuring that our few Ubuntu hosts (mostly used for internal services) were updated
to a safe version of OpenSSL and restarted. At this time I also updated our
vulnerable installations of Chef with the
[latest version](http://www.getchef.com/blog/2014/04/08/release-chef-client-11-12-0-10-32-2/)
released by Opscode. This includes an update of the OpenSSL library shipped with
the omnibus.

On further review of our SSL settings, I realized that our SSL settings regarding
cypher suites were woefully out of date. Following the advice of these two sites:

* https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
* https://www.ssllabs.com/ssltest/index.html

I was quickly able to update our configurations again, ensuring that [perfect
forward secrecy](http://en.wikipedia.org/wiki/Perfect_forward_secrecy) is enabled.
According to ssllabs, our site is now rated A.

### Wednesday

On Wednesday Tori and I paired to continue our follow up of the vulnerability. We
watched [this pleasant video](http://vimeo.com/91425662) explaining the bug and
how absolutely horrendous it is. We then spent the morning analyzing every type
of running process we could find in our infrastructure, basically by running
`ps -ef` on a number of hosts and tracking down everything that either dynamically
linked to libssl or had a version statically compiled.

As a part of our investigation, we found the following:

* several binaries that integrate with various monitoring and alerting tools
  were statically compiled with a dangerous version of OpenSSL
* some of our hosts had a vulnerable version of nodejs installed

As a precaution, we disabled the monitoring daemons (happily, they are tools we
do not rely on in our day-to-day work, just things that are nice to have). We
currently have tickets open with the vendors involved, and will not reënable them
until new versions are provided or we become 100% confident that they are not
vulnerable to the bug.

For the rest of the day we worked on upgrading nodejs to the latest version.

### Thursday

In the morning we continued upgrading nodejs, until we could verify that the only
remaining older versions deployed in our infrastructure were never vulnerable to
the bug. [This post](http://strongloop.com/strongblog/heartbleed-openssl-node-js/)
by Strongloop is a great resource for determining if your version of nodejs is
effected.

This afternoon we had our SSL certificate reissued from our certificate authority.
We carefully deployed it to production and asked for our old certificate to be
revoked. We then distributed the new cert to our CDN.

An unintended consequence was that our old certificate was revoked before our
CDN could deploy the new one, and for a short amount of time in the evening some
users could not log in via the website. We were able to fairly quickly switch our
site to serve necessary assets using our own infrastructure, until our CDN could
roll out the new certificate.

This process was delayed by the fact that only one person had access to our
account with the certificate authority. In our list of things to do, we did not
properly prioritize this nor escalate it loudly enough within the team. In
retrospect we should have reissued a new cert on Tuesday, then orchestrated the
roll-out in any number of different ways. Lack of sleep does crazy things
to a brain.

### What's next?

As an addendum to this issue, there are several things that we've either changed
or that we're going to change:

* find more security lists alerting these sort of problems, and subscribe
* change all passwords we use to connect to other services
* roll all keys used to connect to other services
* hire a security consultant
* document our learnings in project README files, blog posts and our internal
  wiki
* more people in the engineering department will have access to or know how to
  gain accees to our certificate authority

Personally, a habit I promise to adopt is to drop whatever I'm doing an investigate
on the spot whenever I hear a co-worker talk about any new security vulnerability.

We are fortunate in that at this time Wanelo does not store (nor even pass-through)
any truly sensitive personal data. We take the security of our users extremely
seriously, however, and want to ensure that if there does come a time when sensitive
data passes through our servers we are in a position to do so with clear conscience.

-- [Eric](http://wanelo.com/sax "Sax on Wanelo")
