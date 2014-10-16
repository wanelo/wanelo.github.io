---
layout: default
title: 12-Step Program for Scaling Web Applications on PostgreSQL
author: Konstantin Gredeskoul
author_username: kig
---

On Tuesday night this week Wanelo hosted a monthly meeting of [SFPUG](http://meetup.com/postgresql-1/ "San Francisco PostgreSQL User Group")
- San Francisco PostgreSQL User Group, and I gave a talk that presented a summary
to date of Wanelo's performance journey to today. The
presentation ended upo being much longer than I originally anticipated, and went on for an hour and a half. Whoops!
With over a dozen questions near the end, it felt good to share the tips and tricks that we learned while scaling our app.

<iframe src="http://www.slideshare.net/slideshow/embed_code/32478281?rel=0"
  width="825" height="525" frameborder="0" marginwidth="0" marginheight="0" scrolling="no"
  style="border:1px solid #CCC; border-width:1px 1px 0; margin-bottom:5px; max-width: 100%;" allowfullscreen>
</iframe>
<div class="slideshare-title" style="margin-bottom:5px"> <strong>
  <a href="https://www.slideshare.net/kigster/12step-program-for-scaling-web-applications-on-postgresql"
  title="12-Step Program for Scaling Web Applications on PostgreSQL" target="_blank">12-Step
  Program for Scaling Web Applications on PostgreSQL</a> </strong>
  from <strong><a href="http://www.slideshare.net/kigster" target="_blank">Konstantin Gredeskoul</a></strong>
</div>

The presentation [got recorded on video](https://www.youtube.com/watch?v=zsDKaSlzbco),
but it's not a very good quality unfortunately.

In the meantime, you can see the slides for it :)


## 12-Step Program for Scaling Web Applications on PostgreSQL

Are you addicted to slow application performance? Are you ready to make a change? :)
In this presentation, Konstantin Gredeskoul tells the story of how Wanelo grew their
application to serve 3K requests/seconds in just a few months, while keeping latency
low, and tackling each new growth challenge that came their way. He breaks down their
story into a 12-step program for scaling applications atop PostgreSQL. The talk covers
topics ranging from traditional slow query optimization, vertical and horizontal sharding
with PostgreSQL, serializing and buffering frequent writes, as well as using services to
abstract scalability concerns.

With PostgreSQL 9.2 and 9.3 as the primary data store and Joyent Public Cloud as their
hosting environment, the team at Wanelo keeps optimizing the application stack over and
over again using an iterative approach, to keep the latency low, uptime high, and users happy :)

-- [Konstantin](http://wanelo.com/kig "Konstantin on Wanelo")

