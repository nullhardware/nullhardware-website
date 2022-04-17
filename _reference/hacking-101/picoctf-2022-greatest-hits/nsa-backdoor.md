---
title:  picoCTF 2022 - Crypto - NSA Backdoor (500 points)
description: Even brute strength is enough to break this crypto - if you do it right.
hide_index: true
image:
  path: /img/reference/pico-ctf-2022-00000.png
  width: 1024
  height: 512
  thumb: /img/reference/pico-ctf-2022-00000.thumb.png
  alt: Linux Terminal
---

# picoCTF 2022 - Sequences

> **Note**: This article is part of our [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}).

## The Problem

We are given two numbers (`n` and `c`) and some source code for how they are generated. The source code is remarkably similar to the challenge from `Very Smooth`. However there is a slight difference at the end:

```python
n = p * q
c = pow(3, FLAG, n)

print(f'n = {n.digits(16)}')
print(f'c = {c.digits(16)}')
```

Essentially, given {% katex %} n {% endkatex %} and {% katex %} c {% endkatex %} we now have to solve the problem {% katex %} 3^m = c \text{ (mod n)}{% endkatex %} for some {% katex %} m {% endkatex %}. In other words, this is a discrete logarithm problem. In general, discrete logarithms are hard, and are the basis for some strong cryptographic primitives. **There must be something special about this one.**

## Our first break

Right away we know that we can factor `n` into `p` and `q` - this is because we did the same thing in the `Very Smooth` challenge. The hint for that one was use Pollard's p-1 algorithm, so we will do so here again:

```python
import primefac
p = primefac.pollard_pm1(n)
q = n // p
g = mpz(3)
```

With this factorization we break the problem in to two: solve the discrete log `mod p` and `mod q`, and then use chinese remainder theorem to find the logarithm `mod n`.

```python
cp = c % p
cq = c % q
```

## Solving mod p

We know {% katex %} 3^m = cp \text{ (mod p)}{% endkatex %} for some {% katex %} m {% endkatex %}. Since we know at some level we're going to have to solve a discrete logarithm, let's implement the *most basic* (brute force) approach to one:

```python
# BRUTE FORCE!
def dlog_brute(g, h, p, pi):
    '''solves g^x = h (mod p) for some x, where x only takes values in the range [0, pi)'''
    l=[xi for xi in range(pi) if pow(g, xi, p) == h]
    assert len(l) > 0, f"WARNING prime {p}, g={g}, h={h} has NO solutions! Error!"
    assert len(l) < 2, f"WARNING prime {p}, g={g}, h={h} has multiple solutions: {l}. Error!"
    return l[0]
```

What does this function do? All it does is go through `pi` possible values and return the `x` where {% katex %} g^x = h \text{ (mod p)}{% endkatex %}. It also does some sanity checks to make sure there is exactly 1 solution.

Let's see if it works. We will use `g=3`, `m=11`, and `p=17` and see if it can come up with the answer:

```
>>> pow(3,11,17)
7
>>> dlog_brute(3, 7, 17, 17)
11
>>>
```
{:.contains-term}

Why do we have this extra parameter `pi`? It'll become useful later when we don't want to search for every value in `mod p`.

**NOTE**: *If you want this function to go slightly faster for larger values of pi, you can modify `dlog_brute` to avoid re-calculating the powers of `g` each time:*

```python
def dlog_brute(g, h, p, pi):
    '''solves g^x = h (mod p) for some x, where x only takes values in the range [0, pi)'''
    l=[]
    c_power = 1
    for xi in range(pi): # HERES THE BRUTE!
        if c_power == h:
            l.append(xi)
        c_power = c_power*mpz(g) % p # next power of g, just pow(g,xi,p)
    assert len(l) > 0, f"WARNING prime {p}, g={g}, h={h} has NO solutions! Error!"
    assert len(l) < 2, f"WARNING prime {p}, g={g}, h={h} has multiple solutions: {l}. Error!"
    return l[0]
```

**So can we use `dlog_brute` to solve {% katex %} 3^m = cp \text{ (mod p)}{% endkatex %}?**

**Unforunately, No! That is still far too complicated and won't work (in a reasonable amount of time).**

So what good is this function?

Well, it turns out we can break down the problem even further still. There's an algorithm called [Pohlig-Hellman](https://en.wikipedia.org/wiki/Pohlig%E2%80%93Hellman_algorithm). For instances where `p` is a prime number (as is the case here), we can factor `p-1` into its prime factors, compute a result for each of those factors, and then use CRT to give us the answer `mod p`. Due to the nature of the generated values in this problem, we can get away with a very *naive* version of Pohlig-Hellman that does **not** need to handle repeated prime factors.

```python
def naive_pohlig_hellman(h, p, p_factors):
    '''solve g^x === h mod p, when p-1 has prime factors p_factors, assumed multipliticy is 1 (ie: no repeated prime factors).
    Naive implementation of Pohlig-Hellman_algorithm.
    '''
    assert len(p_factors) == len(set(p_factors)), "Repeated prime factor found. The naive form of this algorithm will not work"
    x=[]
    for pi in p_factors:
        gi = pow(g, (p-1)//pi, p)
        hi = pow(h, (p-1)//pi, p)
        x.append(dlog_brute(gi,hi,p,pi))

    X=chinese_remainder(p_factors,x)
    return X
```

Where we are using this basic implementation of CRT (`n` is a list of bases, `x` is a list of remainders)

```python
import math
def chinese_remainder(n, x):
    s = 0
    p = math.prod(n)
    for ni, xi in zip(n, x):
        pi = p // ni
        s += xi * pow(pi, -1, ni) * pi
    return s % p
```

## A new problem

Does `naive_pohlig_hellman` work? The answer is '**Sometimes**'.

*Let's demonstrate with a simple example that doesn't work.*

Let's say `p=23`.

`p-1` has two prime factors: `[2, 11]`.

Let's say we have `m=5` and therefore `c = 3^5 (mod 23) = 13`. Let's now attempt to solve for `m` given `c=13` using `naive_pohlig_hellman`.

```
>>> pow(3, 5, 23)
13
>>> naive_pohlig_hellman(3, 13, 23, [2,11])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "solve.py", line 53, in naive_pohlig_hellman
    dl = dlog_brute(gi,hi,p,pi)
  File "solve.py", line 28, in dlog_brute
    assert len(l) < 2, f"WARNING prime {p}, g={g}, h={h} has multiple solutions: {l}. Error!"
AssertionError: WARNING prime 23, g=1, h=1 has multiple solutions: [0, 1]. Error!
>>>
```
{:.contains-term}

What's going on here? Well, while solving for the prime factor of 2, we hit a problem.

For `pi = 2`:

{% katex display %}
\begin{aligned}
gi &= 3^{\frac{(p-1)}{2}} = 3^{\frac{22}{2}} = 3^{11} \text{ (mod 23)} \\
  &= 1 \text{ (mod 23)} \\
\\
hi &= {13}^{\frac{(p-1)}{2}} = {13}^{\frac{22}{2}} = {13}^{11} \text{ (mod 23)} \\
  &= 1 \text{ (mod 23)} \\
\end{aligned}
{% endkatex %}

**Uhoh!** We are asking `dlog_brute` to solve the equation {% katex %} 1^x = 1 \text{ (mod 2)}{% endkatex %}. That's clearly a problem since both `0` and `1` are equally valid answers to that equation.

So, what happens if we ask `dlog_brute` to solve {% katex %} 3^{x} = 13 \text{ (mod 23)}{% endkatex %} directly?:

```
>>> dlog_brute(3, 13, 23, 23)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "solve.py", line 28, in dlog_brute
    assert len(l) < 2, f"WARNING prime {p}, g={g}, h={h} has multiple solutions: {l}. Error!"
AssertionError: WARNING prime 23, g=3, h=13 has multiple solutions: [5, 16]. Error!
```
{:.contains-term}

**Aha!** So it turns out `m=5` is not the only solution to this problem. {% katex %} 3^{16} = 13 \text{ (mod 23)}{% endkatex %} is a valid solution as well!

What can we do about this? **If we know that the prime factor of 2 is going to have multiple answers, we can remove that prime factor from the list and solve without it.** Then individually use CRT to figure out both valid answers `mod p`.

```
>>> x1=naive_pohlig_hellman(3, 13, 23, [11])
>>> chinese_remainder([(23-1)//2, 2], [x1, 0])
16
>>> chinese_remainder([(23-1)//2, 2], [x1, 1])
5
```
{:.contains-term}

## Solving mod p (Again)

Now that we have a plan, we can solve the discrete logarithm problem (`mod p`) using our naive implementation of Pohlig-Hellman. First we need to factor `p-1` into its prime factors (and for simplicity, we'll sort them as well):

```python
pm1_factors = list(primefac.primefac(p-1))
pm1_factors.sort()
```

Now, we do a quick check to see if we can use these prime_factors directly:

```
>>> naive_pohlig_hellman(3, cp, p, pm1_factors)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "solve.py", line 53, in naive_pohlig_hellman
    dl = dlog_brute(gi,hi,p,pi)
  File "solve.py", line 28, in dlog_brute
    assert len(l) < 2, f"WARNING prime {p}, g={g}, h={h} has multiple solutions: {l}. Error!"
AssertionError: WARNING prime 176473303538524259200554324953336384726672109110665668162293282699973540848874702767584458062843333942678732811932897476909679289489853667242704250498709920215500564359945126566451281262283662096646326724094693217360879121741192532765498098061185923631716696944607478088126741032221004102364580340388512170139, g=1, h=1 has multiple solutions: [0, 1]. Error!
```
{:.contains-term}

**We observe the same problem we saw above with the prime factor 2. Fortunately we know how to deal with it: just remove the prime factor of 2 from the list and deal with it afterwards.**

```python
xp = naive_pohlig_hellman(3, cp, p, pm1_factors[1:])
XP0 = chinese_remainder([(p-1)//2, 2], [xp, 0])
XP1 = chinese_remainder([(p-1)//2, 2], [xp, 1])
assert pow(g,XP0,p) == cp, "XP0 is not solution mod p!"
assert pow(g,XP1,p) == cp, "XP1 is not solution mod p!"
```

So, there we have it. **Two equally valid solutions (mod p) calculated within a couple seconds, even with the most basic and naive of algorithms.**

## Solving mod q

Solving `mod q` is the same, and it also contains two equally valid solutions:

```python
qm1_factors = list(primefac.primefac(q-1))
qm1_factors.sort()
xq = naive_pohlig_hellman(3, cq, q, qm1_factors[1:])
XQ0 = chinese_remainder([(q-1)//2, 2], [xq, 0])
XQ1 = chinese_remainder([(q-1)//2, 2], [xq, 1])
assert pow(g,XQ0,q) == cq, "XQ0 is not solution mod q!"
assert pow(g,XQ1,q) == cq, "XQ1 is not solution mod q!"
```

## Solving mod n

We now have two solutions `mod p` and two solutions `mod q`. For any given pair, we can calculate a unique solution `mod n` using the CRT. Does that mean all pairs are equally valid? **No, they are not - The CRT will always return a result, but that does not mean the result is a valid solution to the discrete logarithm problem (`mod n`). We are only concerned with results that _are_ an actual solution to the discrete logarithm (`mod n`) [For the true solution, the resulting sub-problems (`mod p`) and (`mod q`) will also be true].**

```python
import itertools
from binascii import unhexlify
for a,b in itertools.product([XP0,XP1],[XQ0,XQ1]):
    s=chinese_remainder([p,q],[a,b])
    if pow(g,s,n) == c:
        assert pow(g,s,p) == cp and pow(g,s,q) == cq, "Solution did not satisfy mod p and mod q"
        print(f"Found Message:\nm={s}\n{unhexlify(s.digits(16)).decode()}")
```

Final Code:

```python
# NOTE: required functions (dlog_brute, chinese_remainder, naive_pohlig_hellman) defined above
# read from "output.txt"
vars={}
with open("output.txt") as f:
    for k,v in [x.strip().split(" = ") for x in f]:
        vars[k] = mpz(v,16)

# define variables of interest
n, c = vars["n"], vars["c"]
g = mpz(3)

# factor n into p and q
import primefac
p = primefac.pollard_pm1(n)
q = n // p

# define 2 subproblems, one mod p and one mod q
cp = c % p
cq = c % q

# solve for all solutions mod p (there are 2)
pm1_factors = list(primefac.primefac(p-1))
pm1_factors.sort()
xp = naive_pohlig_hellman(g, cp, p, pm1_factors[1:])
XP0 = chinese_remainder([(p-1)//2, 2], [xp, 0])
XP1 = chinese_remainder([(p-1)//2, 2], [xp, 1])
assert pow(g,XP0,p) == cp, "XP0 is not solution mod p!"
assert pow(g,XP1,p) == cp, "XP1 is not solution mod p!"

# solve for all solutions mod q (there are 2)
qm1_factors = list(primefac.primefac(q-1))
qm1_factors.sort()
xq = naive_pohlig_hellman(g, cq, q, qm1_factors[1:])
XQ0 = chinese_remainder([(q-1)//2, 2], [xq, 0])
XQ1 = chinese_remainder([(q-1)//2, 2], [xq, 1])
assert pow(g,XQ0,q) == cq, "XQ0 is not solution mod q!"
assert pow(g,XQ1,q) == cq, "XQ1 is not solution mod q!"

# try all pairs and see if any are a solution mod m
import itertools
from binascii import unhexlify
for a,b in itertools.product([XP0,XP1],[XQ0,XQ1]):
    s=chinese_remainder([p,q],[a,b])
    if pow(g,s,n) == c:
        assert pow(g,s,p) == cp and pow(g,s,q) == cq, "Solution did not satisfy mod p and mod q"
        print(f"Found Message:\nm='{unhexlify(s.digits(16)).decode()}'")
```

Result:

```
$ python3 solve.py
Found Message:
m='picoCTF{===REDACTED===}'
```
{:.contains-term}

# A note about the solutions

We started by implementing a brute-force solution to the discrete logarithm problem. This allowed us to identify an issue where it was not possible to uniquely determine an answer because there was more than one possible answer. There are several libraries where you can calculate discrete logarithms in a multitude of ways, but that doesn't mean the answer that it gives you back is the *only* answer. We are fortunate in this case because the message itself is quite short, and algorithms that favor "small" answers will give you the message almost immediately. Since `m` is much shorter than both `p` and `q`, the answer to the discrete logarithm problem is the same `mod p` and `mod q`, so it was actually pretty clear which of our solutions was the real `m`.

Head back to the [picoCTF 2022 Greatest Hits Guide]({% link _reference/hacking-101/picoctf-2022-greatest-hits.md %}) to continue with the next challenge.
