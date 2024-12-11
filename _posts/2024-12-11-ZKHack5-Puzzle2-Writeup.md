---
layout: "post"
title:  "ZKHack5 Puzzle 2 Writeup"
date:   2024-12-11 14:44:26 +0100
categories: CTF ZK
---

[Link to puzzle](https://zkhack.dev/zkhackV/puzzleV2.html) here.

---

When beginning puzzles, I like to take stock of all the clues/information given before even looking at the code (I neglected this during the first puzzle when it would have helped tremendously). For this puzzle, we’re told that the LogUp protocol, Protostar and small fields will be important.

- LogUp: I had gone through the second zkhack whiteboard session on [lookups](https://zkhack.dev/whiteboard/s2m3/) which explained the main ideas behind fractional sum-based lookups. 

- Protostar: I am familiar with this mainly in the context of folding, remembered extremely little about the role of lookups here.

- Small fields: Thinking Binius, although it turned out in the code that the field was a little larger than two 😅

If you’re wondering what even are lookups in the first place, I recommend watching [this whiteboard session](https://zkhack.dev/whiteboard/module-six/) before getting into LogUp.

That being said, when I first started reviewing the code for this puzzle, I was rather baffled. The `invalid_witness` value needed to be included in the `witness` array of field elements, but whenever it was included the witness no longer matched the values of the lookup table. 

My first thought was that I needed to make the witness vector overflow, but it turns out rust is pretty good at preventing that. Then, after having looked at the tests given in the `protocol.rs` file, I faffed about for a while to see if I could make enough witness elements match the table by manipulating the values of `m`. This was also quite fruitless.

I decided to go back and revisit my understanding of LogUp and Protostar, particularly the security conditions for their valid use in zk proof systems.

Rewatching the LogUp whiteboard session I struck gold at about 13:15 of the video. It was explained that if an element was included in the witness as many times as the field characteristic, it cancels out. By adding the following code I was able to pass the asserts and solve the puzzle:

```
 /* BEGIN HACK */
    let mut witness = vec![];
    //make the elements of the witness match those of the table
    for i in 0..64 {
        witness.push(FieldElement::new(vec![i], p, irr.clone()));
    }
    //add invalid witness elements as many times as the field characteristic so they cancel
    for _index in 0..p {
        witness.push(invalid_witness.clone());
    }

    let m = vec![FieldElement::new(vec![1], p, irr.clone()); 1 << 6]; 
    /* END HACK */
```

This hack is possible due to the relatively small field size `p = 70937`. Still though, I did not entirely understand what exactly was it about the fractional sums of LogUp that made this possible. 

Looking up what exactly makes this additive cancellation work for our small field, I found it has to do with the definition of the field characteristic. Looking on [wikipedia](https://en.wikipedia.org/wiki/Field_(mathematics) ) we find: 

>In addition to the multiplication of two elements of F, it is possible to define the product n ⋅ a of an arbitrary element a of F by a positive integer n to be the n-fold sum
> `a + a + ... + a (which is an element of F.)`
>If there is no positive integer such that
> `n ⋅ 1 = 0,`
>then F is said to have characteristic 0.

Since we know that we’re given a prime field in this code, we know that any element in the field added to itself as many times as the field characteristic is going to be zero. 

As explained in the whiteboard session video, because the fractional sums involving the elements of `m` and the table on one side of the equality and elements of the witness on the other side are linearly independent, it’s not possible in the LogUp lookup protocol to manipulate the elements of the sums in order to arrive at an equality involving different elements than those in the table and in the witness.

We can verify the linear independence of these fractional sums by checking for different denominators (or if converting to a common denominator, examining the polynomial equality to ensure not all coefficients are zero).

The only way to include the `invalid_witness` is by making it go to zero via additive cancellation, hence why my previous attempts at manipulating the values of `m` were fruitless.