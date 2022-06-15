---
title: Still Hacking
headline: PicoCTF and Beyond
description: >-
  Two years ago I started chipping away at the binary exploitation problems in picoCTF 2018. Now I'm an 'Elite Hacker' on HTB and finished in the Top-10 in picoCTF'22 - Learn how I did it.
author: andrew
image:
  path: /img/generic/hacker-pandemic-00000.png
  width: 2048
  height: 1024
  thumb: /img/generic/hacker-pandemic-00000.thumb.png
  alt: Hacker wearing a mask
cta:
  title: PicoCTF 2022 Write-ups
  cta: "> Get Started"
  url: /reference/hacking-101/picoctf-2022-greatest-hits/
  body: |
    Read about 'Solfire' and others in our detailed write-ups for some of the highest point challenges in PicoCTF'22.
  image:
    path: /img/generic/messy-desk-000000.jpg
    center: '60% 55%'
tags: [hacking]
---

When [last we spoke]({% link _posts/featured/2020-08-11-hacking-the-pandemic-away.md %}) I had started chipping away at the binary exploitation problems in picoCTF 2018. Two years later: I'm now an 'Elite Hacker' on [HTB](http://www.hackthebox.com) and placed in the Top-10 overall in picoCTF 2022. Here's how I did it.

## Hack The Box

I documented my process for progressing from 'Noob' to 'Elite Hacker' (and breaking into the top 3 in Canada) in this [tweet](https://twitter.com/steadmanticore/status/1533255971044085760):

<div>
<blockquote class="twitter-tweet" data-dnt="true"><p lang="en" dir="ltr">Last year I went from n00b to Elite Hacker on <a href="https://twitter.com/hackthebox_eu?ref_src=twsrc%5Etfw">@hackthebox_eu</a> in 3 months. Not going to lie, at first I floundered. I avoided the boxes because of the reputation. I had made steady progress against challenges (particularly pwn), but never got many points. Finally I bit the bullet. <a href="https://t.co/HuMJiNN9iu">pic.twitter.com/HuMJiNN9iu</a></p>&mdash; Andrew Steadman (@steadmanticore) <a href="https://twitter.com/steadmanticore/status/1533255969131532288?ref_src=twsrc%5Etfw">June 5, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

**Full Text Below**:

> Last year I went from n00b to Elite Hacker on @[hackthebox_eu](https://twitter.com/hackthebox_eu) in 3 months. Not going to lie, at first I floundered. I avoided the boxes because of the reputation. I had made steady progress against challenges (particularly pwn), but never got many points. Finally I bit the bullet.

> A friend and I had some time over Christmas and we set a goal of landing in the top 10 in Canada. The first couple boxes weren't too bad, and by then I was hooked. I would wake up every morning at 5 am (sometimes 4am), because that's the only time I could dedicate to this stuff.

> Going from 'pro' to 'elite' hacker was the worst. You've already done all the easy content, and you pray every week that they retire and replace an easy box, because you know you will have to do the new box just to maintain your points, as well as a hard box to gain some ground.

> I spent a ton of time on "Cereal", where you had to trigger a custom deserialization vulnerability, but there was an ip-whitelist, so you had to use a separate XSS vulnerability to inject Javascript so that you could initiate a GET request with the proper authorization header.

> In the end, I topped out at 3rd in Canada, and was able to become 'Elite Hacker' in 3 months.  
>  
> I'm still an 'Elite Hacker', because after that is Guru, and Elite Hacker sounds cooler TBH.

![HTB Canadian Scoreboard](/img/blog/2022-06-15/htb_top_3.png)

## PicoCTF 2022

![PicoCTF'22 Scoreboard](/img/blog/2022-06-15/picoctf2022.png)

2022 was a triumphant return to where it all began: **picoCTF**. Two years later, after hours and hours of practice, my team (Blasto!) placed 6th overall (and was tied for points with second place) after solving all but one challenge. *The worst part*? The challenge I was unable to finish was my wheelhouse: **Binary Exploitation**. 

I had hoped that my experience completing the [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) challenges would better prepare me for this Solana challenge. Unfortunately, although I was able to *steal* enough lamports from the vault account, my mis-understanding of the ownership model meant that I was unable to transfer them to the correct account before the challenge ended. However, if you're curious how to successfully complete this challenge, it's all documented in our [3-part writeup]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}). Our complete list of picoCTF'22 write-ups is available [here]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}):

{% include cta/button.html cta=page.cta %}