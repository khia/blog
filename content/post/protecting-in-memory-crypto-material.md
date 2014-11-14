---
Description: How to protect crypto material from your hosting provider
Keywords:
- crypto
- security
Section: post
Slug: protecting-in-memory-crypto-material
Tags:
- crypto
- security
Title: Protecting in-memory crypto material
Topics:
- Thoughts
- Crypto
- Theory
date: 2014-11-13
---


Protecting in-memory crypto material
====================================

Introduction
------------

Let's say you operate a service which need to sign clients' requests.
You would have to have a private key (signing key) on your server.
Having keys on the server is ok as long as you fully control the server.
You cannot trust the cloud provider if you use one.
Even if a company you use doesn't practice illegal access to information of their customers.
Some individual employees might. So what you would do to protect yourself in this case?


Solution
--------

Unfortunately there is no solution.
If third party controls your hardware. They have access to RAM where the private key is located.
They can use various ways to get the info from there.
If quantum entanglement would be employed in the form of quantum-memory than it might be possible to implement self destruction of crypto key whenever anyone looking at it. Until then there is no hope. However you can mitigate the risk if you employ multiple cloud providers and split private key into individually useless fragments. In this case you could do distributed signing. The correct term for this functionality is "aggregate signatures". There are sequencial aggregate signatures and non-sequencial ones. Sequential usually non-interactive while non-sequential are interactive.

RSA-compatible
--------------

1. ID-Based Sequential Aggregate Signatures by Xiangguo Cheng at el.


Non RSA-compatible
------------------

1. Sequential Aggregate Signatures, Multisignatures, and Veriably Encrypted Signatures Without Random Oracles.

2. Sequential Aggregate Signatures with Short Public Keys: Design, Analysis and Implementation Studies.

Note for myself
---------------

Add bibtex links
