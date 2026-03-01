---
layout: "post"
title:  "Bit Wrangling Adventures"
date:   2026-03-01 17:13:31 +0100
categories: Bit math, Embarrassing stories
---

So um, you guys. Embarrassing story time.

Your girl was asked how she would chop the trailing zeros from a binary number in a job interview. 

Now, as some of you may know, I'm the type of person to forget how to operate a microwave the second my boss comes and watches me work (true story, I do not miss working in restaurants). So far in my life I've been lucky enough to have take-home technical tests for interviews and no one had to watch me do dumb things for ten minutes straight before I got the hang of it. You can imagine, dear reader, the panicky deer-in-headlights stare I probably gave the interviewer as my head went immediately empty upon being asked to perform.

Perhaps because I've been spending all my time with cyclic groups these days, I immediately thought about chopping based on modulo 2 division. Before I could stutter anything about indexes, however, my face was on fire and I realized I couldn't continue with the interview for the shame 😭.

I kept thinking about the problem after that whole mess was over. It seemed simple enough, and I did think my initial idea would work. Turns out, it does:

```
def groovy_mod_way(input_int):
    index = 1

    if input_int == 0:
        return "int is zero, no 1 bits"
    
    while True:
        if input_int % 2**index != 0:
            break
        index += 1

    return index - 1
```

I was pretty sure that was probably not what the guy was looking for though. After a lot of CTFs I have a bit of a spidey sense for when I must be missing the trick to some problem. I checked online, and sure enough there was a bit manipulation trick I had completely missed:

```
def fancy_bit_flipping(input_int):
    if input_int == 0:
        return "int is zero, no 1 bits"
    
    ls1b = input_int & -input_int
    
    return (ls1b).bit_length() - 1
```

Cute. Clever. Still though, if the point was to stay away from libraries/built-ins and implement the algorithm myself I needed to see what Python does under the hood for `.bit_length()`. Turns out, it's just what I did but [wearing a hat](https://python-reference.readthedocs.io/en/latest/docs/ints/bit_length.html): 

>If x is nonzero, then x.bit_length() is the unique positive integer k such that 2**>(k-1) <= abs(x) < 2**k.
    
Now I asked myself what I supposed the interviewer would have asked me had I not melted into the floor like the Elephant's Foot at Chernobyl: can we be **mOaR** efficient about this??? Two more minutes of poking around [nerdy corners of the internet](https://web.archive.org/web/20210726212120if_/https://www.researchgate.net/profile/Keith-Randall/publication/2809440_Using_de_Bruijn_Sequences_to_Index_a_1_in_a_Computer_Word/links/0fcfd5107691fdedff000000/Using-de-Bruijn-Sequences-to-Index-a-1-in-a-Computer-Word.pdf) and I found something very cool, some kind of hybrid approach between my modular division idea and the bitwise AND trick; for fixed-size integers you can use a [de Bruijn sequence](https://en.wikipedia.org/wiki/De_Bruijn_sequence) to index a 1-bit in constant time.

The idea is this: do the bitwise AND operation on your integer to isolate the lowest 1-bit, then multiply by a precomputed de Bruijn constant. This new value needs to then be right-shifted by some value (we'll get to that in a minute) in order to be used as the key to a precomputed lookup table containing the value of the index of the AND-ed bit.

The de Bruijn constant we want is just some particular de Bruijn sequence for length-*n* string given a size-*k* alphabet. It can be any of the possible sequences, it doesn't matter. Since we're doing binary computer stuff, our alphabet size *k = 2* won't ever change, only the length-*n* string. (The only other thing to note is that our lookup table will change if our constant does, despite being the same size as another de Bruijn sequence.)

Now for the really interesting part! The length-*n* we want for our de Bruijn constant is the power of 2 that gives us the size of our integer. 

Think of it like this: de Bruijn sequences are cyclic, with the sequences having order *n*. This means that their subsequences will be cyclic as well. Because of this, there will only ever be one possible occurrence of substrings of length *n* for every de Bruijn sequence. For *n = 3* and *k = 2* where our alphabet *A ={0, 1}*, we'll always find the substrings `000` and `111` exactly once in our sequence.

Let's say we have an 8-bit integer equal to 40. That's `00101000` in binary. We can see that the longest continguous substring of either `0`s or `1`s is 6 digits long. If we do our bit AND trick, we get `00101000 & 11011000 == 00001000` (8 in binary). Now, if we multiply this by the de Bruijn constant `00011101` (29 in binary), we get `8 * 29 == 232` (`11101000`).

Now comes the bit shifting. Because we've got an 8-bit sequence of which we're only interested in the three most significant bits (because *n=3*, remember), we'll need to shift right by `8 - 3 == 5`. Now our value `111` will map to an index value of `3` on a precomputed lookup table. All other possible values for indices 0-7 will be mapped to all other possible three-digit subsequences of the upper bits of our de Bruijn constant multiplied by our isolated 1 bit.

No more need to loop through mod 2 to the power of some increasing index! We've sort of frontloaded the cyclic nature of binary representation with this approach, which I think is beautiful.

Now, as for the applications...uh, some people are really into making efficient [chess games with this technique](https://www.chessprogramming.org/BitScan#Bitscan_by_Modulo). Apparently it's a whole thing. It can also be used to more efficiently [brute-force some types of locks](https://en.wikipedia.org/wiki/De_Bruijn_sequence#Brute-force_attacks_on_locks).

I enjoyed thinking through this problem a lot after that interview, so I consider it a success despite everything. Also, I've started doing those [coding puzzle challenges](https://leetcode.com/) that help you prepare for this type of interview, so hopefully next time will be better. I found an even nerdier, math-ier version of this: [Project Euler](https://projecteuler.net/). They have a really nice Discord too.
