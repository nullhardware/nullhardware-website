---
title: Fixed-Point Sine (and Cosine) for Embedded Systems
headline: 5th Order Polynomial Fixed-Point Sine Approximation
description: >-
  I derive a simple fixed-point approximation to sin (and cos) appropriate for
  embedded systems without dedicated floating-point hardware accurate to within
  0.01% Full-Scale. Presented as dependency-free C code without a lookup-table.
date: 2018-03-28T02:51:18.098Z
author: andrew
draft_img: /img/drafts/sin_graph.png
tags:
  - math
---
I derive and present a simple fixed-point approximation to sin (and cos) appropriate for embedded systems without dedicated floating-point hardware. It is accurate to within ±1/4096 (0.01% Full-Scale). No lookup-table is required, and portable C code is available.

## Background

I originally derived this implementation with an MSP430 in mind. This particular chip has a full integer 32x32 HW MAC, completing most integer multiplications within a couple of cycles. I wanted to avoid the vendor-supplied floating point routines mostly because I had no need for floating point precision. If performance is required, this general approach could be optimized for your particular hardware.

This post is inspired (and a derivative of) the excellent [Another fast fixed-point sine approximation](http://www.coranac.com/2009/07/sines/). Although excellent, I ended up requiring the full 5th order approximation to meet my design specifications.

Since this is a 5th order approximation, I will restrict the derivation discussion to {% katex %}\sin(x){% endkatex %} (which is odd). The cosine can be calculated from the sine with a simple phase shift (see example code).

## Domain

The domain of {% katex %}\sin(x){% endkatex %} is infinite. However, it only provides unique (positive) values within the range {% katex %}x \in [0, \frac{\pi}{2}]{% endkatex %}. All the other outputs can be calculated based on the values within this range and the symmetry of the sine function.

In general, the input to the sine function can be positive or negative. It can be fractional, or it can even be irrational. However, a fixed-point sine should (probably) accept a fixed-point angle as an input. There are many possible choices for the exact mapping to choose, but I will outline a convenient one below.

Whole angles (in degrees) range from {% katex %}0-360{% endkatex %}. An 8-bit integer could at most represent 256 unique values, which is a coarser resolution than a degree, and probably unsuitable for all but the roughest of approximations. The next logical step up is supporting 16-bit inputs.

Signed (two's complement) 16-bit numbers can take values in the range {% katex %}[-32768, 32767]{% endkatex %}. Signed values are nice because angles are quite often represented as positive or negative in many contexts. Unsigned values, on the other hand, would allow us to maximize the bitwise precision of all of our integer multiplications without having to compromise any bits for representing the sign. 

Fortunately, the periodicity of the sine function allows us to have our cake and eat it too. If we strategically choose a value of +32768 to be exactly equal to {% katex %}2\pi{% endkatex %}, then we can write a function with a signature of `int fpsin(int16_t x)` and internally cast the signed value `x` to be unsigned:

```c
fpsin(-32768) = fpsin((uint16_t)-32768) = fpsin(32768)
```
Since we've strategically chosen 32768 to be equal to {% katex %}2\pi{% endkatex %}, `fpsin(-32768) = fpsin(32768)` = {% katex %}\sin(2\pi) = sin(0) = 0{% endkatex %}. Similarly, `fpsin(-1) = fpsin(65535)` = {% katex %}\sin(4\pi - \frac{\pi}{16384}) = \sin(-\frac{\pi}{16384}){% endkatex %} = `fpsin(-1)`.

In essence, by matching the periodicity of the sine function to both the overflow behavior of unsigned 16-bit integers [`65535+1 = 0`] as well the two's complement representation of signed numbers [`(uint16_t)-32768 = +32768`], we can cast the input value to be unsigned without any loss of information.

The end result is that we'd have chose the fixed-point representation of our input angle to be 32768 units/circle.

## The approximation

According to the derivation in [Another fast fixed-point sine approximation](http://www.coranac.com/2009/07/sines/), the {% katex %}5^\text{th}{% endkatex %} order polynomial that minimizes the root-mean-squared approximation error over the region {% katex %}x \in [0, \frac{\pi}{2}]{% endkatex %} or {% katex %}z = \frac{x}{\frac{\pi}{2}} = \frac{2x}{\pi} \in [0, 1]{% endkatex %} is:

{% katex display %}
\begin{aligned}
sin_5(x) &= a_5 z - b_5 z^3 +  c_5 z^5 \text{, where}\\
a_5 &= 4 \bigg(\frac{3}{\pi} - \frac{9}{16}\bigg), \\
b_5 &= 2 a_5 - \frac{5}{2}, \\
c_5 &= a_5 - \frac{3}{2}
\end{aligned}
{% endkatex %}

The only tricky part is to write this equation in terms of integer multiplications. My intended target is (mostly) portable C code. As a result, I will write my multiplications such that the multiplicand, multiplier, and product are all of the same type (uint32_t). This is a strange way to write multiplications, and really only makes "sense" in C. Further optimizations specialized to your MAC HW (if you have one) are probably worth while if speed is required. Since we want a fixed-point value as an output, we multiply the actual value by a fixed-point multiplier {% katex %}2^a{% endkatex %}.

{% katex display %}
\begin{aligned}
fpsin_5(x) &=\bigg( a_5 z - b_5 z^3 + c_5 z^5 \bigg) 2^a
\end{aligned}
{% endkatex %}

Because {% katex %}2\pi = 32768{% endkatex %}, we know {% katex %}\frac{\pi}{2} = 8192 = 2^n{% endkatex %} where {% katex %}n = 13{% endkatex %}. Therefore, we can write {% katex %}z{% endkatex %} as {% katex %}\frac{y}{2^n}{% endkatex %}.

{% katex display %}
\begin{aligned}
 &= z \bigg( a_5 - z^2 \bigg[ b_5 - z^2 c_5 \bigg] \bigg) 2^a\\
 &= \frac{y}{2^n} \bigg( a_5 - \frac{y^2}{2^{2n}} \bigg[ b_5 - \frac{y^2}{2^{2n}} c_5 \bigg] \bigg) 2^a \\
 &= y 2^{-n} \bigg( a_5 - y^2 2^{-2n} \bigg[ b_5 - y^2 2^{-2n} c_5 \bigg] \bigg) 2^{a} \\
 &= y 2^{-n} \Bigg( a_5 - y 2^{-n} y 2^{-n} \Bigg[ b_5 - y 2^{-2n} c_5 y \Bigg] \Bigg) 2^{a}
\end{aligned}
{% endkatex %}

To maximize precision, we need each of our multiplications to occupy as much of our chosen product type (`uint32_t`) as we can. To this end, we introduce scaling factors {% katex %}2^p{% endkatex %}, {% katex %}2^q{% endkatex %}, and {% katex %}2^r{% endkatex %}, being careful that they each cancel out (so as to not affect the output).

{% katex display %}
\begin{aligned}
 &= y 2^{-n} \Bigg( a_5 - y 2^{-n} y 2^{-n} \Bigg[ b_5 - 2^{-r} y 2^{-2n} 2^r c_5 y \Bigg] \Bigg) 2^{a} \\
  &= y 2^{-n} \Bigg( a_5 - 2^{-p} y 2^{-n} y 2^{-n} \Bigg[ 2^p b_5 - 2^{-r} y 2^{-2n} 2^{r+p} c_5 y \Bigg] \Bigg) 2^{a} \\
  &= y 2^{-n} \Bigg( 2^q a_5- 2^{q-p} y 2^{-n} y 2^{-n} \Bigg[ 2^p b_5 - 2^{-r} y 2^{-n} 2^{r+p-n} c_5 y \Bigg] \Bigg) 2^{a-q}
\end{aligned}
{% endkatex %}

After scaling, we can re-define the constants: {% katex %}A_1 = 2^{q}a_5 {% endkatex %}, {% katex %} B_1 = 2^{p} b_5 {% endkatex %}, and {% katex %} C_1 = 2^{r+p-n} c_5 {% endkatex %}.

{% katex display %}
\begin{aligned}
  &= y 2^{-n} \Bigg( A_1 - 2^{q-p} y 2^{-n} y 2^{-n} \Bigg[ B_1 - 2^{-r} y 2^{-n} C_1 y \Bigg] \Bigg) 2^{a-q} 
\end{aligned}
{% endkatex %}

*Note*: All but the innermost multiplication by {% katex %}y{% endkatex %} has been preceded by a multiplication by {% katex %}2^{-n}{% endkatex %}. Since the largest value of {% katex %}y{% endkatex %} is {% katex %}2^n{% endkatex %}, we know {% katex %}y 2^{-n} x \le x{% endkatex %} (where {% katex %}x{% endkatex %} is any number).

We can now try and maximize each multiplication, working from the inner-most multiplication outward, such that each product (assuming {% katex %}y{% endkatex %} is {% katex %}2^n = 2^{13}{% endkatex %}) is exactly 32 bits. We also scale the fixed-point result to be as large as possible while introducing error only in the least significant bit. The end result is:

{% katex display %}
\begin{aligned}
  a &= 12 \\
  p &= 32 \\
  q &= 31 \\
  r &= 3 \\
  \text{and:}\\
  A_1 &= 3370945099 \\
  B_1 &= 2746362156 \\
  C_1 &= 292421
\end{aligned}
{% endkatex %}

## Code

The only remaining task is to convert the above equation into C code. Some tricks are done to determine the sign of the output, as well as to keep the input range from {% katex %}[0,2^{13}]{% endkatex %}, but otherwise everything is pretty straightforward.

```c
/*
Implements the 5-order polynomial approximation to sin(x).
@param i   angle (with 2^15 units/circle)
@return    16 bit fixed point Sine value (4.12) (ie: +4096 = +1 & -4096 = -1)

The result is accurate to within +- 1 count. ie: +/-2.44e-4.
*/
int16_t fpsin(int16_t i)
{
    /* Convert (signed) input to a value between 0 and 8192. (8192 is pi/2, which is the region of the curve fit). */
    /* ------------------------------------------------------------------- */
    i <<= 1;
    uint8_t c = i<0; //set carry for output pos/neg

    if(i == (i|0x4000)) // flip input value to corresponding value in range [0..8192)
        i = (1<<15) - i;
    i = (i & 0x7FFF) >> 1;
    /* ------------------------------------------------------------------- */

    /* The following section implements the formula:
     = y * 2^-n * ( A1 - 2^(q-p)* y * 2^-n * y * 2^-n * [B1 - 2^-r * y * 2^-n * C1 * y]) * 2^(a-q)
    Where the constants are defined as follows:
    */
    enum {A1=3370945099UL, B1=2746362156UL, C1=292421UL};
    enum {n=13, p=32, q=31, r=3, a=12};

    uint32_t y = (C1*((uint32_t)i))>>n;
    y = B1 - (((uint32_t)i*y)>>r);
    y = (uint32_t)i * (y>>n);
    y = (uint32_t)i * (y>>n);
    y = A1 - (y>>(p-q));
    y = (uint32_t)i * (y>>n);
    y = (y+(1UL<<(q-a-1)))>>(q-a); // Rounding

    return c ? -y : y;
}

//Cos(x) = sin(x + pi/2)
#define fpcos(i) fpsin((int16_t)(((uint16_t)(i)) + 8192U))
```

## Testing

We can verify the fixed-point approximation using floating-point routines and comparing.

```c
int main(void)
{
    int32_t max = 0, min = 0;
    for(uint16_t i = 0; i <= 32768; ++i)
    {
        int32_t s = lround(4096*sin(2*M_PI * i / 32768));
        int16_t s5d = fpsin(i);
        int32_t err = s - s5d;
        if(err > max)
            max = err;
        if(err < min)
            min = err;
        printf("The value of %i is %i - compare %i, diff : %i\n", i, s5d, s, err);
    }
    
    printf("min: %i max: %i\n", min, max);
    return 0;
}
```

Running this code verifies that the error is no larger than {% katex %}\pm 1{% endkatex %} counts of our chosen fixed-point representation (ie: 1/4096). Since the range of sine is {% katex %}[-1, +1]{% endkatex %}, this error is only 0.01% of the entire range ({% katex %}\frac{1/4096}{1 - ^-1} = 0.0001 = 0.01\%{% endkatex %}).