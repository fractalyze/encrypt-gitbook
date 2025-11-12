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
    Point powerOfP = p;
    if (k & 1) {             
        result = result + powerOfP; 
    }
    k >>= 1; 
    while (k > 0) {
        powerOfP = powerOfP.Double(); 
        if (k & 1) {             
            result = result + powerOfP; 
        }
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
powerOfP = powerOfP.Double()  // 1 double : powerOfP = 2p
result += powerOfP            // 1 add    : result = 2p

// 2nd while loop:
powerOfP = powerOfP.Double()  // 1 double : powerOfP = 4p
result += powerOfP            // 1 add    : result = 6p

= 2 adds + 2 doubles
```

> Written by [Ashley Jeong](https://app.gitbook.com/u/PNJ4Qqiz7kSxSs58UIaauk95AO43 "mention") of Fractalyze
