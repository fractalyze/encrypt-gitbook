# ElGamal Encryption

## ElGamal Encryption

Given a **cyclic group** $$\mathbb{G}$$ of order $$q$$ and its **generator** $$g$$, the encryption and decryption process for a message $$M$$ is defined as follows.

### **ElGamal Encryption Process**

#### $$\mathsf{ElGamal.Setup}() \rightarrow (x, y)$$**:**

* Select a **secret key** $$x$$ randomly from the range $$1 \leq x < q-1$$.
* Compute the **public key** $$y$$ as follows:

$$
y = g^x
$$

#### $$\mathsf{ElGamal.Encrypt}(M, y) \rightarrow (c_1, c_2)$$:

* Map the message $$M$$ to an element $$m \in \mathbb{G}$$.
* Select a **random value** $$k$$ from the range $$1 \leq k < q-1$$.
* Compute the **shared secret** $$s$$:

$$
s = y^k
$$

* Generate the ciphertext $$(c_1, c_2)$$ as follows:

$$
c_1 = g^k, \quad c_2 = m \cdot s
$$

#### $$\mathsf{ElGamal.Decrypt}(c_1, c_2, x) \rightarrow M$$:

* Recover the shared secret $$s$$ (This can only be done by the owner of $$x$$):&#x20;

$$
s = c_1^x = (g^k)^x = (g^x)^k = y^k
$$

* Retrieve the original message $$m$$:

$$
m = c_2 \cdot s^{-1} = (m \cdot s) \cdot s^{-1}
$$

* Convert $$m$$ back to the message $$M$$.

## References

* [https://en.wikipedia.org/wiki/ElGamal\_encryption](https://en.wikipedia.org/wiki/ElGamal_encryption)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
