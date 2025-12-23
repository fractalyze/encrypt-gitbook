# R1CS

## Introduction

The Rank 1 Constraint System (R1CS) is one of the common arithmetization techniques used today. In short, it uses 3 matrices to constrain two inputs to the gate and 1 output of the gate.

$$
\bm{Az}\cdot \bm{Bz}\stackrel{?}=\bm{Cz}
$$

If the equation above is valid for a given $$\bm{A},\bm{B},\bm{C}$$ constraint matrices, we consider $$\bm{z}$$ a valid witness (also known as trace).

So how do we get these constraint matrices for a target computation? For the sake of simplicity, let's say we want to prove that we know a solution to $$x^3+x+5=35$$.

## **Flattening to gates**

First we convert our $$x^3 + x + 5$$ program into simple statements which typically look like $$x=y$$ or $$x=y \oplus z$$. So $$x^3 + x + 5$$ can be flattened to:

```
sym_1 = x * x  
y = sym_1 * x  
sym_2 = y + x  
out = sym_2 + 5
```

## **Gates to R1CS**

In this step we convert the equations above into an R1CS matrix representation. At each row of R1CS matrices, we have 3 row vectors $$\bm{a},\bm{b},\bm{c}$$ which we multiply with column vector $$\bm{z}$$. This solution vector $$\bm{z}$$ should satisfy

$$
\bm{z}\cdot \bm{a} \times \bm{z} \cdot \bm{b} - \bm{z} \cdot \bm{c} = 0\\ 
\bm{z}\cdot \bm{a} \times \bm{z} \cdot \bm{b} = \bm{z}\cdot \bm{c}
$$

Each vector's length is equal to the number of variables in our program plus 1 for constant 1. So in the case above, we have $$\char`\~one,x, sym_1, sym_2, sym_3, \char`\~ out$$ so we need vectors of size 6. We need the extra constant $$1$$ or identity multiplier to deal with situations involving addition in our scheme as well as to add constants. So in total for our case above we have:

```
index     0    1.   2     3.     4.   5.
varname ~one.  x. ~out  sym_1    y.  sym_2
```

The reason for having three vectors is due to the nature of the constraints R1CS is designed to represent. Remember, in circuit computation, we have conceptual gates. These gates have a left input, a right input, and one output. So, basically, the $$\bm{a}$$ vector represents the left input, the $$\bm{b}$$ vector represents the right input, and the $$\bm{c}$$ vector represents the output.

1. To express $$sym_1 = x \cdot x$$, what we need to say is input 1 and input 2 is $$x$$ and result is $$sym_1$$

$$
\bm{a} = (0, 1, 0, 0, 0, 0)\\ 
\bm{b} = (0, 1, 0, 0, 0, 0) \\ 
\bm{c} = (0, 0, 0, 1, 0, 0)
$$

2. Similarly, to express $$y = sym_1\cdot x$$:

$$
\bm{a} = (0, 0, 0, 1, 0, 0)\\ 
\bm{b} = (0, 1, 0, 0, 0, 0)\\ 
\bm{c} = (0, 0, 0, 0, 1, 0)
$$

2. For $$sym_2 = (x + y) \cdot 1$$, we have:

$$
\bm{a} = (0, 1, 0, 0, 1, 0)\\ 
\bm{b} = (1, 0, 0, 0, 0, 0)\\ 
\bm{c} = (0, 0, 0, 0, 0, 1)
$$

4. And for $$\char`~out = (sym_2 + 5) \cdot 1$$, we have:

$$
\bm{a} = (5, 0, 0, 0, 0, 1)\\ 
\bm{b} = (1, 0, 0, 0, 0, 0)\\ 
\bm{c} = (0, 0, 1, 0, 0, 0)
$$

This gives us the R1CS matrix representation:

```
A  
[0, 1, 0, 0, 0, 0]  
[0, 0, 0, 1, 0, 0]  
[0, 1, 0, 0, 1, 0]  
[5, 0, 0, 0, 0, 1]
B  
[0, 1, 0, 0, 0, 0]  
[0, 1, 0, 0, 0, 0]  
[1, 0, 0, 0, 0, 0]  
[1, 0, 0, 0, 0, 0]
C  
[0, 0, 0, 1, 0, 0]  
[0, 0, 0, 0, 1, 0]  
[0, 0, 0, 0, 0, 1]  
[0, 0, 1, 0, 0, 0]
```

So by knowing $$\bm{z} = [1, 3, 35, 9, 27, 30]$$, you prove that you know the solution since it satisfies all the constraints.

## References

* &#x20;[https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
