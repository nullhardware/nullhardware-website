---
title:  picoCTF 2022 - Greatest Hits
description: Writeups for picoCTF 2022 Challenges.
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Challenges

This time around we broke out of the Binary Exploitation category. We eventually solved *everything* (and broke into the global top 10 during the competition). This page is a *greatest-hits* compilation of the more difficult/interesting challenges and our approach to solving them.

**NOTE: stay tuned as this page is actively being updated with more challenges.**

## Getting Started

Note: For most challenges all you need as a linux machine (a Kali VM or WSL2 is fine) with python3.

## List of Challenges

### 1. [Sequences]({% link _reference/hacking-101/picoctf-2022-greatest-hits/sequences.md %}) (**Crypto** - *400 Points*)

>
```
Traceback (most recent call last):
  File "sequences.py", line 48, in <module>
    sol = m_func(ITERS)
  File "sequences.py", line 19, in m_func
    return 55692*m_func(i-4) - 9549*m_func(i-3) + 301*m_func(i-2) + 21*m_func(i-1)
  File "sequences.py", line 19, in m_func
    return 55692*m_func(i-4) - 9549*m_func(i-3) + 301*m_func(i-2) + 21*m_func(i-1)
  File "sequences.py", line 19, in m_func
    return 55692*m_func(i-4) - 9549*m_func(i-3) + 301*m_func(i-2) + 21*m_func(i-1)
  [Previous line repeated 995 more times]
  File "sequences.py", line 14, in m_func
    if i == 0: return 1
RecursionError: maximum recursion depth exceeded in comparison
```
> Solving crypto problems through the power of linear algebra!  
> [> Read More]({% link _reference/hacking-101/picoctf-2022-greatest-hits/sequences.md %})
{:.contains-term}

### 2. [Solfire]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %}) (**Binary Exploitation** - *500 Points*)

>
```
$ file solfire.so
solfire.so: ELF 64-bit LSB shared object, eBPF, version 1 (SYSV), dynamically linked, not stripped
```
> Our three-part series covers reversing and exploiting a Solana smart-contract to steal over 50,000 lamports. This challenge had less than 10 solves during the picoCTF 2022 competition.  
> [> Read More]({% link _reference/hacking-101/picoctf-2022-greatest-hits/solfire.md %})
{:.contains-term}


### 3. [Live Art]({% link _reference/hacking-101/picoctf-2022-greatest-hits/live-art.md %}) (**Web Exploitation** - *500 Points*)

>
```
$ docker run --rm  -p 3000:3000 picoctf2022-liveart
Pre-bundling dependencies:
  react
  react-dom
  react-router-dom
  peerjs
  react/jsx-dev-runtime
(this will be run only when your dependencies or config have changed)
  vite v2.8.6 dev server running at:
  > Local:    http://localhost:3000/
  > Network:  http://172.17.0.2:3000/
  ready in 501ms.
```
> This React website contains a hidden XSS vulnerability that we'll need to figure out in order to steal the flag.  
> [> Read More]({% link _reference/hacking-101/picoctf-2022-greatest-hits/live-art.md %})
{:.contains-term}
