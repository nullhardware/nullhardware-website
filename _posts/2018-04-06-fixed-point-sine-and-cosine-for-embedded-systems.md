---
title: Fixed-Point Sine (and Cosine) for Embedded Systems
headline: 5th Order Polynomial Fixed-Point Sine Approximation
description: >-
  I derive a simple fixed-point approximation to sin (and cos) appropriate for embedded systems without dedicated floating-point hardware. The implementation is accurate to within 0.01% of Full-Scale and is presented as dependency-free C code (without a lookup-table).
author: andrew
image:
  path: /img/generic/sin-wave-000000.png
  width: 2048
  height: 1024
  thumb: /img/generic/sin-wave-000000.thumb.png
  alt: Graph of the Sine (sin) function, which is approximated using an integer based 5th order polynomial.
tags: [math, code]
---
Here is a simple fixed-point approximation to sin (and cos) appropriate for embedded systems without dedicated floating-point hardware. It is accurate to within ±1/4096 (0.01% Full-Scale). No lookup-table is required, and portable C code is available.

## Background

> I originally derived this implementation with a MSP430 in mind. With a full integer 32x32 HW MAC, but no floating point unit, fixed-point routines made the most sense for the math. If performance is required, this general approach could be optimized for your particular hardware.

This post is inspired by, and a derivative of, the excellent [Another fast fixed-point sine approximation](http://www.coranac.com/2009/07/sines/). However, that page neglects the complete derivation of the 5<sup>th</sup> order implementation, so I was forced to derive my own.

As a 5<sup>th</sup> order approximation, I will restrict the derivation discussion to {% katex %}\sin(x){% endkatex %} (which is odd). The cosine can be calculated from the sine with a simple phase shift (see example code).

## Domain

The domain of {% katex %}\sin(x){% endkatex %} is infinite. However, it only provides unique (positive) values within the range {% katex %}x \in [0, \frac{\pi}{2}]{% endkatex %}. All the other outputs can be calculated based on the values within this range and the symmetry of the sine function.

In general, the input to the sine function can be positive, negative, fractional, or even irrational. However, a fixed-point sine function should (most likely) accept a fixed-point angle as an input.

Whole angles (in degrees) range from {% katex %}0-360{% endkatex %}. An 8-bit integer could at most represent 256 unique values, which is a coarser resolution than a degree, and probably unsuitable for all but the roughest of approximations. The next logical step up is supporting 16-bit inputs.

Signed (two's complement) 16-bit numbers can take values in the range {% katex %}[-32768, 32767]{% endkatex %}. Signed values are nice because angles are quite often represented as positive or negative in many contexts. Unsigned values, on the other hand, would allow us to maximize the bitwise precision of all of our integer multiplications without having to compromise any bits for representing the sign. We will keep these factors in mind while deriving our approximation.

## The Approximation

According to the [source material](http://www.coranac.com/2009/07/sines/), the 5<sup>th</sup> order polynomial that minimizes the root-mean-squared approximation error over the region {% katex %}x \in [0, \frac{\pi}{2}]{% endkatex %} or {% katex %}z = {x}/{\frac{\pi}{2}} = {2x}/{\pi} \in [0, 1]{% endkatex %} is:

{% katex display %}
\begin{aligned}
sin_5(x) &= a_5 z - b_5 z^3 +  c_5 z^5 \text{, where}\\
a_5 &= 4 \bigg(\frac{3}{\pi} - \frac{9}{16}\bigg), \\
b_5 &= 2 a_5 - \frac{5}{2}, \\
c_5 &= a_5 - \frac{3}{2}
\end{aligned}
{% endkatex %}

## The Errors

The 5<sup>th</sup> order approximation given above has a maximum error of approximately 1.9e-4. Therefore, if we choose our fixed-point output to have 12 fixed-point bits (corresponding with a resolution of 2.4e-4), we could expect to contain the approximation error to the least significant digit.

*In other words, our fixed-point approximation is simply the above result multiplied by the fixed-point multiplier {% katex %}2^{12} = 2^{a}{% endkatex %}.*

Now we must reconsider our input domain. If our output resolution is {% katex %}{1}/{4096}{% endkatex %}(2.4e-4), it follows that we should choose an input quantization that restricts the errors arising from quantization to be strictly less than the output resolution.

The slope of the sine function is steepest in the region around {% katex %}x=0{% endkatex %}. Similarly, the slope of our polynomial approximation is also  steepest at {% katex %}x=0{% endkatex %}. If we want to limit the error caused by input quantization, this is the region that we want to consider.

For small values near {% katex %}0{% endkatex %}, {% katex %}\sin(z) \approx z{% endkatex %}. However, this is **not** true for our polynomial approximation, which is approximately {% katex %}a_5 z{% endkatex %}. Consider the approximate error term {% katex %}e(z) = 4096 \big( a_5 z - z\big){% endkatex %}:

| {% katex %}z{% endkatex %} | {% katex %}e(z){% endkatex %}|
| --- | --- |
| {% katex %}2^{-14}{% endkatex %} | 0.142 |
| {% katex %}2^{-13}{% endkatex %} | 0.285 |
| {% katex %}2^{-12}{% endkatex %} | 0.570 |
| {% katex %}2^{-11}{% endkatex %} | 1.140 |

If we want to limit our error to the least significant digit with our chosen fixed-point multiplier, we should consider quantizing the input to be **at least** 13 fixed-point bits.

## Quantization

Now we must rewrite this equation in terms of only integer multiplications and simple bit shifts. My intended target is C code, so I will write my multiplications such that the multiplicand, multiplier, and product are all of the same type (`uint32_t`). This is a strange way to write multiplications, and it does not optimize for the register level timing or complete resolution capability of HW MACs. Further optimizations specialized to your hardware are probably worth while if speed is required.

{% katex display %}
\begin{aligned}
fpsin_5(x) &=\bigg( a_5 z - b_5 z^3 + c_5 z^5 \bigg) 2^a
\end{aligned}
{% endkatex %}

Because {% katex %}z \in [0, 1]{% endkatex %}, we can write {% katex %}z{% endkatex %} as {% katex %}{y}/{2^n}{% endkatex %} where {% katex %}y \in [0,2^n]{% endkatex %}.

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

We can now try and maximize each multiplication, working from the inner-most multiplication outward, such that each product is exactly 32 bits. We also scale the fixed-point result to be as large as possible while introducing error only in the least significant bit. The end result is:

{% katex display %}
\begin{aligned}
  a &= 12 \\
  n &= 13 \\
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

With the hard part done, it is a relatively simple matter to write the above equation as C code. Some tricks are used to pre-determine the sign of the output and then use unsigned multiplication throughout, but otherwise everything is pretty straightforward.

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