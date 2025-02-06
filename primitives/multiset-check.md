# Multiset Check

MultiSet Check is the process of verifying whether two MultiSets contain the same elements. Unlike a regular set, a MultiSet can include duplicate values. Let's explain this concept with three examples below.

1. The two sets below have the same elements, although in different orders, so they are equal.

$$
A = \{1, 2, 3, 4\} \\
B = \{1, 2, 3, 2\}
$$

2. The two sets below have different sizes, so they aren’t equal.

$$
A = \{1, 2, 3\} \\
B = \{1,2,3,2\}
$$

3. The two sets below are the same size, but only $$A$$ contains the value 4, so they aren’t equal.

$$
A = \{1,2,3,4\} \\
B = \{1,2,3,2\}
$$

So, how can we determine if two arbitrary sets satisfy MultiSet equality? At first glance, it might seem sufficient to simply multiply all the values together, but would this really work?

$$
A = \{a_0, a_1 , \dots, a_{n-1}\} \\
B = \{b_0, b_1, \dots, b_{n-1}\} \\
\prod_{i=0}^{n-1}a_i \stackrel{?}= \prod_{i=0}^{n-1}b_i
$$

In fact, there can be numerous counterexamples to the equation as shown below.

$$
A = \{1,2,3,4\} \\
B = \{1,1,4,6\}
$$

To probabilistically verify the above safely, we can use the [Schwartz–Zippel lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma) to construct polynomials and perform checks as follows. Using the same example above, if the size of the entire field is $$|\mathbb{F}|$$, the calculations match on only 4 values on both sides, making it probabilistically secure.

$$
\prod_{i=0}^{n-1}(X - a_i) \stackrel{?}= \prod_{i=0}^{n-1}(X - b_i)
$$

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
