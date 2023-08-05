---
title: "A bandersnatch implementation in Lambdaworks"
date: 2023-08-06T00:34:58+02:00
draft: false
---

During Lambda ZK Week [by LambdaClass](https://lambdaclass.com/), we ([Dragan](https://twitter.com/DraganP_test97), [0xKitetsu](https://twitter.com/0xKitetsu), [phklive](https://twitter.com/phklive) and myself) built an implementation of [the Bandersnatch elliptic curve](https://eprint.iacr.org/2021/1152.pdf) using [Lambdaworks](https://github.com/lambdaclass/lambdaworks/)

It was very fun and I learned so much during this event, so let me introduce you to my learnings :)

# What is Bandersnatch?

Bandersnatch is an [elliptic curve](https://en.wikipedia.org/wiki/Elliptic_curve) built over the BLS12-381 scalar field. If you don't know what elliptic curves, I can only advice [this article!](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/)

So Bandersnatch is an EC, and has been created to use the [GLV endomorphism](https://www.iacr.org/archive/eurocrypt2009/54790519/54790519.pdf), enabling faster scalar multiplication compared to other EC such as Jubjub.

Bandersnatch is used in the first implementations of [Verkle Tries](https://github.com/gballet/go-verkle), which makes it a pretty important curve, since it could be used in Ethereum! 

# Lambdaworks 

Lambdaworks is a library of crypto primitives made by LambdaClass, made as an alternative to arkworks (another crypto primitives library).

The library already has some EC implemented such as BLS12-381 or Baby-Jubjub, but Bandersnatch was not implemented :(

# Our hack

During the hackathon, at first we wanted to do a Verkle Tries implementation using Lambdaworks library, but we quickly realized that the library was missing some primitives (IPA, Pedersen, Bandersnatch...)

So we decided to just focus on one part, bandersnatch!

None of us ever implemented something like this, so we had to learn everything from scratch, but we got good help from a lot of people at the hackathon!

![Help at the hackathon](/posts/bandersnatch-lambda-zk-week/zklambda-hackathon.jpeg)

Lambdaworks was really easy to navigate and add new curves was something not very hard to do, but the hardest part of the hack was to actually understand what to do!

At the end of the hackathon, we managed to create a first implementation of the curve and make a [PR to Lambdaworks](https://github.com/lambdaclass/lambdaworks/pull/513)

Unfortunately, we didn't had enough time to implement GLV endomorphism.

# Next steps

We really want to take time to understand and implement the GLV endomorphism as well, and make benchmarks against arkworks implementation! 

We're planning to continue our implementation soon, to be continued in another article then :D 
