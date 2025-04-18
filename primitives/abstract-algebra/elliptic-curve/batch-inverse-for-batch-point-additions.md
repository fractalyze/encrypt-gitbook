# Batch Inverse for Batch Point Additions

## Motivation

The addition of two elliptic curve points $$P=(x_P,y_P)$$ and $$Q=(x_Q,y_Q)$$ to receive $$R = P+Q = (x_R, y_R)$$ must be done like so ([See here for more information on elliptic curves](./)):

$$
m = \frac{y_Q - y_P}{x_Q - x_P} \\
x_R=m^2-x_P-x_Q\\
y_R=m(x_P-x_R)-y_P
$$

We see that the calculation of the slope $$m$$ always includes the field inverse operation of $$(x_Q-x_P)^{-1}$$, but we know that field inverse operations are expensive. Thus, when computing multiple point additions at once, we want to find a way to only require a singular field inverse operation in total. This can simply be done with our previously established concept of [Batch Inverse](../group/batch-inverse.md).

## Algorithm Process

We apply[ Batch Inversion](../group/batch-inverse.md) to our point additions. Suppose we have two sets of $$n$$ total points as $$\{P_0, P_1, \dots ,P_{n-1}\}$$ and $$\{Q_0, Q_1, \dots ,Q_{n-1}\}$$ where we are attempting to obtain the sums $$\{R_0, R_1, \dots ,R_{n-1}\}$$ where $$R_i = P_i + Q_i$$.

Note that the points $$P_i$$ and $$Q_i$$ are defined as follows:

$$
\begin{align*}
P_i &= (x_{P_i}, y_{P_i}) \\
Q_i &= (x_{Q_i}, y_{Q_i}) \\
\end{align*}
$$

As per definition, for a single point addition, computing $$m_i$$ requires computing $$(x_{Q_i}-x_{P_i})^{-1}$$.

Thus, we define the following instead:

$$
\begin{align*}
\lambda_i &= x_{Q_i} - x_{P_i} \\
(x_{Q_i} -x_{P_i})^{-1} &= \frac{1}{\lambda_i} =\frac{\prod_{j\neq i}\lambda_j}{\prod_{k=0}^{n-1}\lambda_k}
\end{align*}
$$

With this definition, the only field inverse operation required across all $$(x_{Q_i} -x_{P_i})^{-1}$$ calculations is $$(\prod_{k=0}^{n-1} \lambda_k)^{-1}$$. As $$\prod_{k=0}^{n-1} \lambda_k$$ is the total product of all $$\lambda$$ values, it only needs to be calculated once across all inverses $$(x_{Q_i} -x_{P_i})^{-1}$$. Therefore, we now require the inverse of only one value when computing multiple point additions at once.&#x20;

> Note that the numerator $$\prod_{j\neq i}\lambda_j$$ will introduce new field multiplication operations, but this is fine as the cost of these multiplications is far less than computing more field inverse operations.

Refer to the [Computational Complexity Section](../group/batch-inverse.md#computational-complexity) of [Batch Inversion](../group/batch-inverse.md) to learn more about the total cost.

## References

* [https://hackmd.io/1mpavmFmQNWrahBi8mHBjQ](https://hackmd.io/1mpavmFmQNWrahBi8mHBjQ)
* [https://hackmd.io/@tazAymRSQCGXTUKkbh1BAg/Sk27liTW9#Optimization-1-Batch-Affine-for-Point-Addition](https://hackmd.io/@tazAymRSQCGXTUKkbh1BAg/Sk27liTW9#Optimization-1-Batch-Affine-for-Point-Addition)

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of [A41](https://www.a41.io/)
