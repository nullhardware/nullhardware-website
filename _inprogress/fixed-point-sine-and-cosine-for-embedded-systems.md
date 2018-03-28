---
title: Fixed-Point Sine (and Cosine) for Embedded Systems
headline: 5th order polynomial fixed-point sine approximation
description: >-
  I derive a simple fixed-point approximation to sin (and cos) appropriate for
  embedded systems without dedicated floating-point hardware accurate to within
  0.01% Full-Scale.
date: 2018-03-28T02:51:18.098Z
author: andrew
draft_img: /img/drafts/DSC05499_500_501_fused-4.jpg
tags:
  - math
---
I will derive and present a simple fixed-point approximation to sin (and cos) appropriate for embedded systems without dedicated floating-point hardware. It is accurate to within ±1/4096 (0.01% Full-Scale).

## Background

I originally derived this implementation with an MSP430 in mind. This particular chip had a full integer 32x32 HW MAC, completing most integer multiplications within a couple of cycles. I wanted to avoid the vendor-supplied floating point routines mostly because I had already managed to avoid them entirely in an near-complete project and felt no reason to introduce them now. If performance is required, this general approach should be optimized for your particular hardware.

This post is inspired (and a derivative of) the excellent [Another fast fixed-point sine approximation](http://www.coranac.com/2009/07/sines/). Although excellent, for my application, I ended up requiring the full 5th order approximation to meet the design specifications.

Since this is a 5th order approximation, I will restrict the derivation discussion to {% katex %}\sin(x){% endkatex %} (which is odd). The cosine can be calculated from the sine with a simple phase shift.

## Domain

The domain of {% katex %}\sin(x){% endkatex %} is infinite. However, it only provides unique (positive) values within the range {% katex %}x \in [0, \frac{\pi}{2}]{% endkatex %}. All the other outputs can be calculated based on the values within this range and the symmetry of the sine function.

In general, the input to the sine function can be positive or negative. It can be fractional, and it can even be irrational. A fixed-point sine should (probably) accept a fixed-point angle as an input. There are many possible choices for the exact mapping to choose, but I will outline a convenient one below.

Whole angles (in degrees) range from 0-360. An 8-bit integer could at most represent 256 unique values, which is a coarser resolution than a degree, and probably in-suitable for all but the roughest of approximations. The next logical step up is allowing up to 16-bit inputs.

Signed (two's complement) 16-bit numbers can take values in the range {% katex %}[-32768, 32767]{% endkatex %}. Signed values are nice because angles are quite often represented as positive or negative in many contexts. Unsigned values, on the other hand, would allow us to maximize the bitwise precision of all of our integer multiplications without having to compromise any bits for representing the sign. 

Fortunately, the periodicity of the sine function allows us to have our cake and eat it too. If we strategically choose a value of +32768 to be exactly equal to {% katex %}2\pi{% endkatex %}, then we can write a function with a signature of `int fpsin(int16_t x)` and internally cast the signed value `x` to be unsigned like this: `uint16_t x_ = (uint16_t)x`, then we'd have the following:

```c
fpsin(-32768) = fpsin((uint16_t)-32768) = fpsine(32768)
```

And since we've strategically chosen 32768 to be equal to {% katex %}2\pi{% endkatex %}, `fpsin(-32768) = fpsine(32768) = `{% katex %}\sin(2\pi) = sin(0) = 0.{% endkatex %}
