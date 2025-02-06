# PLONK

## Introduction

Main reference: [Understanding PLONK by Vitalik Buterin](https://vitalik.eth.limo/general/2019/09/22/plonk.html)

The **PLONK arithmetization** is done by representing the target computation $$P$$ as a circuit with logic gates for addition and multiplication, and converting it into a system of equations where the variables are the values on all the wires.

<figure><img src="../../.gitbook/assets/Pasted image 20240530183311.png" alt=""><figcaption><p>Figure 1. Example circuit for <span class="math">x^3+x+5=35</span>. Source: <a href="https://vitalik.eth.limo/general/2019/09/22/plonk.html">https://vitalik.eth.limo/general/2019/09/22/plonk.html</a></p></figcaption></figure>

On the gates and wires, we have two types of constraints:

1. **Gate constraints** - equations between wires attached to the same gate, e.g. $$a_1\cdot b_1=c_1$$
2. **Copy constraints** - claims about equality of different wires anywhere in the circuit, eg. $$a_0=a_1=b_1$$ or $$c_0=a_1$$. We will need to create a structured system of equations, which will ultimately reduce to a very small number of polynomial equations, to represent both constraints.

## **Gate Constraints**

In PLONK, each gate equation is of the following form (Note: $$L$$ = left, $$R$$ = right, $$O$$ = output, $$M$$ = multiplication, $$C$$ = constant, and $$(a,b)$$ is the left and right input to the gate):

$$
𝑄_{𝐿}𝑎+𝑄_𝑅𝑏+(𝑄_𝑂)𝑐+(𝑄_𝑀)𝑎𝑏+𝑄_𝐶=0
$$

So for an addition gate, we set:

$$
Q_L=1,Q_R=1,Q_O=-1,Q_M=0,Q_C=0
$$

Meanwhile, for a multiplication gate, we set:

$$
Q_L=0, Q_R=0, Q_O=-1,Q_M=1, Q_C=0
$$

As you can probably see, there's nothing so far forcing the output of one gate to be the same as the input of another gate (what we call "copy constraints").

Now, if we make these equations for each gate, we have a system of many equations which we can reduce to 1 big polynomial using [QAP](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649):

$$
Q_L(x)a(x) + Q_R(x)b(x) + Q_O(x)c(x) + Q_M(x)a(x)b(x) + Q_C(x) = 0
$$

With this, we can check gate constraints.

## **Copy Constraints**

Let us define $$a(i), b(i), c(i)$$ to be input and output wires of $$i$$-th gate (i.e. $$a$$ is the left input, $$b$$ is the right input, $$c$$ is the output). Now we want to make sure the gates are wired correctly. As an example, let's try to check if $$a(1)\stackrel{?}=a(3)$$.

The key idea is we introduce an _accumulator_ of the values and show that even if we switch the value at $$a(1)\leftrightarrow a(3)$$, the accumulation result is the same.

First, consider a polynomial $$X$$ with the evaluation form:

$$
X  : \{(0,0),(1,1), (2,2), (3,3)\} \\
$$

And a simple accumulator $$p(x)=p(x-1)\times(X(x)+a(x))$$. With this construction, if the values of $$a$$ are $$\{1,2,3,2\}$$,

$$
p(4)=(0+1) \times (1+2)\times(2+3)\times(3+2)=75.
$$

Now, if we define another polynomial $$X'$$ with the evaluation form: $$\{(0,0), (1,3),(2,2),(3,1)\}$$, then for a similarly defined $$p'$$, the value of $$p'(4)$$ would be also $$75$$ since $$a(1)=a(3)$$. Conversely, if $$p'(4)=p(4)$$, does it mean that $$a(1)=a(3)$$? Yes. But I think this is not necessarily true for more complex checks.

Now, suppose we want to prove a copy constraint that spans different columns (i.e. $$a(1)=c(2)=b(3)$$). In this case, we construct $$X_a(x)=x, X_b(x)=4+x, X_c(x)=4+x$$. Then, one possible evaluations of the $$X'_a,X'_b,X'_c$$ over $$x\in\{0,1,2,3\}$$ can be:

$$
X'_a:\{0,7,2,3\}\\
X'_b:\{4,5,6,10\}\\
X'_c:\{8,9,1,11\}
$$

With this, it suffices to check if $$p_a(4)\times p_b(4)\times p_c(4)=p'_a(4)\times p'_b(4)\times p'_c(4)$$.

NOTE: The actual accumulator polynomial is defined as $$p(x)=p(x-1)\times(v_1 + X(x)+v_2\times a(x))$$ where $$v_1$$ and $$v_2$$ are randomly sampled. TODO: Elaborate on the reasons behind this construction.

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/qkmdeDQ0VghI3poGEvfJmiZECAg1 "mention") from A41
