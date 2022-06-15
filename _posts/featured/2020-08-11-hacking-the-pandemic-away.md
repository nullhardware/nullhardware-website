---
title: Hacking the Pandemic Away
headline: PicoCTF 2018 in Lockdown
description: >-
  I've spent the last couple of months of lockdown hacking my way through the PicoCTF 2018 capture the flag challenges - and I'm sharing what I've learned. Every single Binary Exploitation challenge has been cracked, including the ones requiring heap exploits. Can you figure out how I did it?
author: andrew
image:
  path: /img/generic/hacker-pandemic-00000.png
  width: 2048
  height: 1024
  thumb: /img/generic/hacker-pandemic-00001.thumb.png
  alt: Hacker wearing a mask
cta:
  title: PicoCTF 2018 Reference Guide
  cta: "> Get Started"
  url: /reference/hacking-101/picoctf-2018-binary-exploits/
  body: |
    Step-By-Step instructions to learn Binary Exploitation by practicing on the PicoCTF 2018 challenges.
  image:
    path: /img/generic/messy-desk-000000.jpg
    center: '60% 55%'
tags: [hacking]
---
I've spent the last couple of months of the pandemic lockdown hacking my way through the [PicoCTF 2018](https://2018game.picoctf.com/) capture the flag challenges - and I'm sharing what I've learned. Every single Binary Exploitation challenge has been cracked, including the ones requiring heap exploits. Can you figure out how I did it?

**TLDR**: Jump straight to our [PicoCTF 2018 BinExp Guide]({% link _reference/hacking-101/picoctf-2018-binary-exploits.md %}).

## Background

[PicoCTF](https://picoctf.com/) is a free series of challenges around computer security, ostensibly aimed at high-school students. They become progressively more challenging as you make your way through them. If you're just starting out with hacking/computer security, then it's a great way to brush up your skills.

As of *right now*, the 2018 competition is still ~~open~~ (**UPDATE: As of 2021 the picoCTF 2018 servers are offline**), and you can register today and start working your way through the challenges. Even 2 years after registration originally opened, my relatively new account has already cracked into the top 300 (out of ~64,000). There are many different categories of challenges: Forensics; General Skills; Reversing; Crypto; Web; and my favourite, *Binary Exploitation*.

Binary Exploitation, in short, is taking an existing binary (possibly with or without the source code), and giving it inputs that force it to execute unexpected code [usually something that launches a shell like `/bin/sh`]. In a CTF, the hacker uses this control to access a "flag" (text file) that they otherwise wouldn't have access to. By submitting a copy of the flag to the system, the hacker demonstrates that they have "cracked" the challenge, and earns the points associated with it.

## But Why?

What I **love** about Binary Exploitation challenges is that in order to do them, you have to have a very good understanding of exactly how your computer executes machine code. This is one of the *few* things that really forces you to understand your platform's [ABI](https://en.wikipedia.org/wiki/Application_binary_interface), well beyond even most software developers (unless you write compilers). In the process, you become acutely aware of the "standard" functions that are dangerous and/or often used insecurely - a mistake you are unlikely to repeat. Furthermore, exploiting these binaries requires a certain aptitude at reading and writing assembly code - a valuable skill in both computer security and software development.

These challenges are fun, force you to think outside of the box, and teach you a whole bunch about coding, memory management, and computer security.

## Writeups

As a resource for others, I'm starting to document everything I've learned under the ["Hacking-101"]({% link reference/index.html %}#hacking-101) section of our general [Reference]({% link reference/index.html %}) page.

[Click Here]({% link _reference/hacking-101/picoctf-2018-binary-exploits.md %}) for access to the complete rundown of the Binary Exploitation challenges from PicoCTF 2018. There are 20 challenges in total, and over the next couple weeks I'll be writing something about each of them.

To really learn something, you should [register](https://2018game.picoctf.com/) for PicoCTF 2018 and attempt them by yourself. But, for those looking for inspiration or who are stuck on a problem, our reference guide will teach you how you can approach the problems without spoiling everything.

{% include cta/button.html cta=page.cta %}