# ZK

## Circuit&#x20;

A circuit can be represented as follows:

$$
C(x: \{ \dots \},\ w: \{ \dots \})
$$

Here, $$x$$ denotes the **public input**, and $$w$$ denotes the **witness** (i.e., private input).

The circuit outputs $$C(x, w) = 1$$ if the conditions are satisfied; otherwise, it returns $$0$$.

### **Example**

Suppose you want to create a circuit that checks whether you know the square root of a value $$X$$ without revealing the square root itself. The circuit can be expressed as:

$$
C(x: \{ X \},\ w: \{ y \}):\quad y^2 \stackrel{?}{=} X
$$

This verifies that $$y$$ is a valid square root of the public input $$X$$.
