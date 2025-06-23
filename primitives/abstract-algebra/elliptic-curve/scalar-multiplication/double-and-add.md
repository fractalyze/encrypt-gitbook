---
description: Reduction of scalar multiplication into double and add operations
---

# Double-and-add

## Motivation

Additions (including doubles) are cheap for elliptic curve points. As such, we try to reduce scalar multiplication (or elliptic curve points multiplied by a scalar) to a series of additions.

## Algorithm Process

In practice, Double-and-add is implemented as follows:

```cpp
// Scalar multiplication: Double-and-Add
Point ScalarMultiply(Point p, int k) {
    Point result; // point at infinity
    while (k > 0) {
        if (k & 1) result += p;
        result = result.Double();
        k >>= 1;
    }
    return result;
}
```

For example, let's take a look at 6:

$$
6 = \underbrace{0}_{2^0}\underbrace{1}_{2^1}\underbrace{1}_{2^2}
$$

```
k = 6
result = 0;

// 1st while loop:
result = result.Double() // 1 double => 0

// 2nd while loop:
result += p              // 1 add    => p
result = result.Double() // 1 double => 2p

// 3rd while loop:
result += p              // 1 add    => 3p
result = result.Double() // 1 double => 6p

= 2 adds + 3 doubles
```

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of A41
