---
title:  picoCTF 2022 - crypto - Sequences (400 points)
description: Optimize a slow recursive function so that it's fast (and doesn't blow up) through the power of linear algebra!
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Sequences

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}).

## The Problem

The **good news** is that we are given some python code that will print out the flag when run. The **bad news**? It won't run. The code is very slow and quickly hits python's recursion limit.

```python
# This will overflow the stack, it will need to be significantly optimized in order to get the answer :)
def m_func(i):
    if i == 0: return 1
    if i == 1: return 2
    if i == 2: return 3
    if i == 3: return 4

    return 55692*m_func(i-4) - 9549*m_func(i-3) + 301*m_func(i-2) + 21*m_func(i-1)
```

## The technique

Let's load up python, define the function, and try out some values to see what happens:

```
>>> m_func(0)
1
>>> m_func(1)
2
>>> m_func(2)
3
>>> m_func(3)
4
>>> m_func(4)
37581
>>> m_func(5)
873142
```
{:.contains-term}

If you've seen this kind of thing before, you'd recognize that this function describes a linear system, where the next state depends on a linear combination of a finite number of previous states. With some thought, you can write a matrix equation that describes the output {% katex %} y_n {% endkatex %} as the multiplication of a matrix {% katex %} M {% endkatex %} and an input state {% katex %} x_{n-1} {% endkatex %}.

{% katex display %}
\begin{aligned}
y_n &= M * x_{n-1}
\end{aligned}
{% endkatex %}

We know that the output is uniquely defined by the 4 known initial values: 1,2,3,4. We can consider these the initial input state: {% katex %} x_0 {% endkatex %}. Given this initial state (represented as a column vector), how should we represent the next output state, {% katex %} y_1 {% endkatex %}?

{% katex display %}
\begin{aligned}
y_1 &= M
  \begin{bmatrix}
   1 \\
   2 \\
   3 \\
   4 \\
  \end{bmatrix}
\end{aligned}
{% endkatex %}

For simplicity, we'd like to keep the dimensionality of the output state the same as the input state. Also, we'd like to be able to chain the multiplications so that the output state {% katex %} y_1 {% endkatex %} can be used directly as the input state of the next iteration (ie: {% katex %} x_1 = y_1 {% endkatex %}). With these considerations in mind, we can write the 4x4 matrix {% katex %} M {% endkatex %}:

{% katex display %}
\begin{aligned}
y_1 &= M x_0 \\
    &= \begin{bmatrix}
    0 & 1 & 0 & 0\\
    0 & 0 & 1 & 0\\
    0 & 0 & 0 & 1\\
    55692 & -9549 & 301 & 21
  \end{bmatrix} \begin{bmatrix}
    1\\
    2\\
    3\\
    4
  \end{bmatrix} \\
  &= \begin{bmatrix}
    2\\
    3\\
    4\\
    37581
  \end{bmatrix}
\end{aligned}
{% endkatex %}

We see the "oldest" value (1) drops out of the state, the other values shift up to replace it, and a new value is calculated based on the 4 previous values. Similarly, we can now calculate the next state {% katex %} y_2 {% endkatex %} by multiplying (on the left) by {% katex %} M {% endkatex %} again (recall {% katex %} x_1 = y_1 {% endkatex %}).

{% katex display %}
\begin{aligned}
y_2 &= M x_1 = M y_1 \\
    &= \begin{bmatrix}
    0 & 1 & 0 & 0\\
    0 & 0 & 1 & 0\\
    0 & 0 & 0 & 1\\
    55692 & -9549 & 301 & 21
  \end{bmatrix} \begin{bmatrix}
    2\\
    3\\
    4\\
    37581
  \end{bmatrix} \\
  &= \begin{bmatrix}
    3\\
    4\\
    37581 \\
    873142
  \end{bmatrix}
\end{aligned}
{% endkatex %}

Interestingly, because we know {% katex %} x_1 = y_1 {% endkatex %}, and we know {% katex %} y_1 = M x_0 {% endkatex %}, we can also write {% katex %} y_2 {% endkatex %} as a function of {% katex %} x_0 {% endkatex %} directly.

{% katex display %}
\begin{aligned}
y_2 &= M x_1 = M y_1 = M \left( M x_0 \right) = M^2 x_0\\
    &= M^2 \begin{bmatrix}
    1\\
    2\\
    3\\
    4
  \end{bmatrix}
\end{aligned}
{% endkatex %}

## Strategy

Ok, we can write this as a linear system in terms of some matrix, *so what*?

The problem is that we have to calculate `m_func(x)` for some **really** large `x` (specifically `2e7`). Given the matrix representation, maybe there's a way to refactor the equation so that the math becomes really simple, even for large numbers?

To start with, we've already observed that by multiplying {% katex %} x_0 {% endkatex %} by {% katex %} M {% endkatex %} once we start to see some new values, and that by chaining repeated multiplications by {% katex %} M {% endkatex %} we can calculate even more values. Let's consider the value of `m_func(4)`, how might we calculate that?

One way is to observe that the result (37581) is the last row of the multiplication of {% katex %} M {% endkatex %} and {% katex %} x_0 {% endkatex %}.

{% katex display %}
\begin{aligned}
y_1 &= M x_0 \\
  &= \begin{bmatrix}
    2\\
    3\\
    4\\
    37581
  \end{bmatrix}
\end{aligned}
{% endkatex %}

But, to keep it simple, since we are attempting to evaluate `m_func(4)` we can also look at the result of {% katex %} M^4 \; x_0 {% endkatex %} and see what we get:

{% katex display %}
\begin{aligned}
y_4 &= M^4 \; x_0 \\
  &= \begin{bmatrix}37581\\873142\\29776743\\529489144\end{bmatrix}
\end{aligned}
{% endkatex %}

Aha! The **first** row of {% katex %} M^4 \; x_0 {% endkatex %} contains what we want. Another way to write that is as a multiplication by a row vector that pulls out only the first term:

{% katex display %}
\begin{aligned}
\text{m\_func(n)} &= \begin{bmatrix}1 & 0 & 0 & 0\end{bmatrix} M^n \; x_0 \\
&= \begin{bmatrix}1 & 0 & 0 & 0\end{bmatrix} M^n \begin{bmatrix}
    1\\
    2\\
    3\\
    4
  \end{bmatrix} 
\end{aligned}
{% endkatex %}

**Now what? Well, now we need to find some way to calculate {% katex %} M^n \; {% endkatex %} efficiently.**

## Background Info

Given a (diagonalizable) square matrix, it [turns out](https://en.wikipedia.org/wiki/Eigendecomposition_of_a_matrix) we can factor {% katex %} M {% endkatex %} into a product of a diagonal matrix {% katex %} D {% endkatex %} (containing eigenvalues) and another matrix {% katex %} P {% endkatex %} (containing eigenvectors):

{% katex display %}
\begin{aligned}
M &= P D P^{-1}
\end{aligned}
{% endkatex %}

How does this help? Well, because we know {% katex %} P^{\tiny{-1}} P = I {% endkatex %} (ie: it cancels out to unity), the powers of {% katex %} M {% endkatex %} simplify:

{% katex display %}
\begin{aligned}
M^2 &= M M = P D P^{-1} \quad P D P^{-1} \\
    &= P D \; I \; D P^{-1} = P D^2 \; P^{-1} \\
M^3 &= M M M = P D P^{-1} \quad P D P^{-1} \quad P D P^{-1} \\
    &= P D I D I D P^{-1} = P D^3 \; P^{-1} \\
M^n &= \text{...} = P D^n \; P^{-1} \\
\end{aligned}
{% endkatex %}

Therefore, our function becomes:

{% katex display %}
\begin{aligned}
\text{m\_func(n)} &= \begin{bmatrix}1 & 0 & 0 & 0\end{bmatrix} M^n \; x_0 \\
&= \underbrace{\begin{bmatrix}1 & 0 & 0 & 0\end{bmatrix} P}_{L} \quad D^n \quad \underbrace{P^{-1} \begin{bmatrix}
    1\\
    2\\
    3\\
    4
  \end{bmatrix}}_{R}
\end{aligned}
{% endkatex %}

Where {% katex %} D {% endkatex %} is a diagonal matrix, so {% katex %} D^n {% endkatex %} is trivial to calculate as it is simply each diagonal element taken to the n-th power. The vector {% katex %}L{% endkatex %} comes from multiplying {% katex %}\begin{bmatrix}1 & 0 & 0 & 0\end{bmatrix}{% endkatex %} and {% katex %}P{% endkatex %} and the vector {% katex %}R{% endkatex %} comes from multiplying {% katex %}P^{\tiny{-1}}{% endkatex %} and {% katex %}x_0{% endkatex %}.

## Solution

Let's break out some python and calculate {% katex %} P {% endkatex %} and {% katex %} D {% endkatex %}. Because we are interested in **exact** solutions where matrix terms are represented by fractions (as opposed to numerical floating points), we will be using the sympy package:

```python
from sympy import *
M=Matrix([[0,1,0,0],[0,0,1,0],[0,0,0,1],[55692,-9549,301,21]])
P,D = M.diagonalize()
Pi=P**-1
print(f"M = {M}\nP*D*P^-1 = {P*D*Pi}\n")
L=Matrix([[1,0,0,0]])*P # pre-multiply by [1,0,0,0]
R=Pi*Matrix([[1],[2],[3],[4]]) #post-multiply by [1;2;3;4]
f=1/gcd(tuple(R)) # pull out the gcd
R=f*R
print(f"M**n = {L} * {D}**n * {R} / {f}")
```

Which, when run, results in the following output:

```
M = Matrix([[0, 1, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1], [55692, -9549, 301, 21]])
P*D*P^-1 = Matrix([[0, 1, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1], [55692, -9549, 301, 21]])

M**n = Matrix([[-1, 1, 1, 1]]) * Matrix([[-21, 0, 0, 0], [0, 12, 0, 0], [0, 0, 13, 0], [0, 0, 0, 17]])**n * Matrix([[-1612], [981920], [-1082829], [141933]]) / 42636
```
{:.contains-term}

Looking at that result, we can recognize that the diagonal matrix structure means that the corresponding terms on the left and right vectors can only influence their respective diagonal term. You can therefore directly write the result as:

{% katex display %}
\begin{aligned}
\text{m\_func(n)} &= \frac{1612*(-21)^n + 981920*(12)^n - 1082829*(13)^n + 141933*(17)^n}{42636}
\end{aligned}
{% endkatex %}

Or, in python

```python
f=lambda n: (1612*((-21)**int(n)) + 981920*((12)**int(n)) - 1082829*((13)**int(n)) + 141933*((17)**int(n)))//42636
```

Now, let's verify:

```
>>> f=lambda n: (1612*((-21)**int(n)) + 981920*((12)**int(n)) - 1082829*((13)**int(n)) + 141933*((17)**int(n)))//42636
>>> f(0)
1
>>> f(1)
2
>>> f(2)
3
>>> f(3)
4
>>> f(4)
37581
>>> f(5)
873142
```
{:.contains-term}

So far so good! That means we're done, *right*?

## The problem

Well, we have a function that appears to work, and it directly computes the answer for any input term without any recursion or memory problems. So, is it good? Well.... running the function with the input `2e7` does give you a result, **eventually**, but it still takes well over a minute. *Also, don't attempt to print it to your screen (because it is very large, converting it to a string a printing it out takes even longer)*.

Ideally we could calculate the answer mod 10^10000, which is all the internals of the decryption function care about.  This would mean that we could potentially use smaller numbers during our power calculations. However, the division in our solution causes a problem, because it has no multiplicative inverse mod 10^10000.

After some testing, it seems the slowest part of the function seems to be calculating the powers of our eigenvalues. ie: `(-21)**int(2e7)` takes a *really* long time.

When it comes to optimizing python, I've learned that the only way to go faster by writing more python is to significantly change the algorithm. Now, it could be that python's algorithm for calculating large powers is unreasonable, but that seems somewhat unlikely. It could also be that python's native representation of integers is inefficient for large integers, and this seems somewhat more likely.

When I asked about this problem to [others](https://sb418.net/ctfs/), it was suggested to me that I investigate the `mpz` type from the gmpy2 library. Here's what the documentation has to say:
> The mpz type is compatible with Python’s built-in int/long type but is significanly *(sic)* faster for large values. 

That sounds perfect. Let's see if it helps:

```
>>> from gmpy2 import mpz
>>> f=lambda n: (1612*(mpz(-21)**int(n)) + 981920*(mpz(12)**int(n)) - 1082829*(mpz(13)**int(n)) + 141933*(mpz(17)**int(n)))//42636
>>> from timeit import timeit
>>> timeit(lambda: f(2e7), number=1)
1.1134801999942283
```
{:.contains-term}

**Boom** - that was quick! Just swap out `m_func` for our new function and grab your flag.

```python
ITERS = int(2e7)

# ... rest of code goes here ...

# OPTIMIZED
from gmpy2 import mpz
f=lambda n: (1612*(mpz(-21)**int(n)) + 981920*(mpz(12)**int(n)) - 1082829*(mpz(13)**int(n)) + 141933*(mpz(17)**int(n)))//42636

if __name__ == "__main__":
    sol = f(ITERS)
    decrypt_flag(sol) # prints picoCTF{===REDACTED===}
```

Head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/Hacking-101/picoCTF-2022-Greatest-Hits.md %}) to continue with the next challenge.
