# Group

## Definition

A group is defined as a set $$G$$ closed under a binary operation ★ and is usually instantiated as$$<G, ★>$$.

If our operation ★ = $$+$$ or is addition, we call this an **Additive Group.**

If our operation ★ = $$*$$ or is multiplication, we call this a **Multiplicative Group.**

### Properties

Groups hold **4 properties**:

1. Closure aka “closed”
   1. $$x★y$$ is in $$G$$, for all $$x,y$$ in $$G$$
2. Associativity
   1. $$(x★y)★z = x★(y★z)$$, for all $$x,y,z$$ in $$G$$
3. Identity
   1. There exists a single “$$e$$” in $$G$$ such that $$x★e = e★x = x$$
4. Inverse
   1. There exists an $$x^{-1}$$ in $$G$$ such that $$x★x^{-1} = x^{-1}★x = e$$ , where “$$e$$” refers to the “$$e$$” of the identity property

> If the Commutative property is also valid ($$x★y = y★x,$$ for all $$x, y$$ in $$G$$), then the group is called an “**Abelian Group.”**

## Examples

1. $$<\mathbb{Z}$$, $$+>$$ additive for the set of all integers ✅ (valid group)
   1. closure ✅
   2. associativity ✅
   3. identity $$a + 0 = 0 + a = a$$ ✅
   4. inverse $$a + (-a) = (-a) + a = 0$$ ✅
2. $$<\mathbb{Z}$$, $$*>$$ multiplicative for the set of all integers ❌ (not a valid group)
   1. closure ✅
   2. associativity ✅
   3. identity $$a1 = 1a = a$$ ✅
   4. inverse $$a * \frac{1}{a} = \frac{1}{a} * a = 1$$, but $$\frac{1}{a}$$ does not necessarily exist in $$\mathbb{Z}$$ ❌
