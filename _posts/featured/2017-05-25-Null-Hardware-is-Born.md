---
description: "We're fed up with proprietary designs, so we're developing Open Source Hardware and Software for Embedded Systems. The journey starts now."
tags: [starting]
image:
  path: /img/generic/messy-desk-000000.jpg
  center: '60% 55%'
  width: 2048
  height: 1024
  thumb: /img/generic/messy-desk-000000.thumb.jpg
cta:
  title: "Don't be shy. SHARE."
  body: |
    And don't forget to Tweet us and/or write to our Wall. We're reaching out to help everyone that does.
  image:
    path: /img/generic/messy-desk-000000.jpg
    center: '60% 55%'
---

For too long technical solutions have evaded beginner makers and hackers.
{: .lead}

> *Obscure forum posts detailing initialization sequences divined through trial and error.*

> *Five year old comments on a vendor website detailing a critical oversight on a datasheet.*

These scenarios are far too common in our industry, but that's not the worst of it. There's also the well-intentioned (but naive) makers who post *bad*, *misleading*, and *outright wrong* information. 

For years we ignored it. We're a team of [professional embedded developers](/about/). We were trained to filter out the good from the bad, to use our google-fu to seek out the answers that we sought. When we wanted something, we figured it out ourselves. We had access to expensive diagnostic equipment. We had simulation software and electronic design automation tools. But it caught up with us. It was constant work, but we have nothing more to show for it than some project in the rear-view mirror. No ground was gained, and we fought the same battles over and over again.

# Frakking Toasters...

The Arduino was born back in 2003. After almost 15 years, it stands as a testament to what open hardware can accomplish.

From toasters to drones, there's nothing an Arduino hasn't done. And we've seen you do it. Your videos, your project writeups, and your excited tweets about your latest Adafruit or Sparkfun order.

The truth is, we love making as much as you do, but we're jealous that you're having so much fun.

# Bringing Back the Love

To rekindle our love affair with electronics, we're planning a few new projects. We'll outline some of them below, but they're really just a starting point. A trajectory. A direction.

What we really want is a conversation, a conversation with you:

> We yearn to hear about your projects, pitfalls, and platforms.

*Do you ever yearn?*  We do.

Our website doesn't have comments working yet, so **Tweet** us [@nullhardware](https://twitter.com/nullhardware) or **write on our [Facebook Wall](https://facebook.com/nullhardware)** and tell us what platforms you're working with.

We'll DM back *every single person* who does. We'll even help you get your current project from a drawing board dream to a reality.

## Potential Upcoming Projects *(Work In Progress)*

 - **Standalone Serial Data (SPI/I2C/UART) Logger**: Log bi-directional (TX/RX) communications onto an SD-card. Great for in-the-field debugging of hard-to-catch problems.

 - **Low-Overhead Debug Logger**: Arduino developers are all very familiar with using `Serial.println()` to debug. However, doing so can actually change the timing of important events, which is infuriating when you are trying to fix infrequent and or timing related bugs. We're planning a low-overhead logging mechanism that minimizes serial traffic and instruction overhead to the bare essentials. Its design is similar in spirit to a production debug and logging mechanism that [Andrew](/about/#andrew) worked with at Blackberry over 10 years ago.

 - **Eddystone Beacons**: We've ordered several cheap Chinese modules capable of broadcasting BLE/Eddystone packets. We're going to try and hack them to do our bidding and unleash them on an unsuspecting public. Will we be successful? Will we still have any friends when all is said and done? Stay tuned to find out.

{% include cta/share.html cta=page.cta %}