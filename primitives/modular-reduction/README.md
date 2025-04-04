---
description: 'Presentation: https://youtu.be/2vC5fDOdYck'
---

# Modular Reduction

## Introduction

To calculate modular addition, $$(A+B) \mod q$$, we can simply&#x20;use a piecewise function:

$$
(A + B) \mod q = \begin{cases}
A + B &\text{if }A + B < q \\
A + B − q &\text{if }A + B ≥ q \end{cases}
$$

While modular adder is simple and easy to implement,&#x20;modular multipliers are trickier. The standard algorithm for&#x20;modular multiplication uses [trial division](https://en.wikipedia.org/wiki/Trial_division), which is inefficient, not scalable, and difficult to implement in hardware&#x20;architecture. The most popular workaround uses&#x20;[Barrett](https://app.gitbook.com/o/vlMwaGV8Ukn8S4gDaGYp/s/rwz1ZAZJtK5FHz4Y1esA/~/changes/55/primitives/modular-multiplication/barrett-reduction) or [Montgomery](https://app.gitbook.com/o/vlMwaGV8Ukn8S4gDaGYp/s/rwz1ZAZJtK5FHz4Y1esA/~/changes/55/primitives/modular-multiplication/montgomery-reduction) reduction based multiplication algorithm. There is also some specialized multiplication algorithms like Karatsuba-Barrett Algorithm etc.
