---
title: 'Code Contracts &#8211; Part 3 : Runtime Checker &#038; Basic Contracts'
author: ammeep
layout: post
permalink: /2011/01/07/201117code-contracts-part-3-runtime-checker-basic-contracts-html/
comments: true
dsq_thread_id:
  - 952352657
categories:
  - Code
tags:
  - Code Contracts
image: /images/featured/contracts.jpeg
summary: The last in the series
---
{% img center /images/posts/code-contracts/runtime.png%}

**Post Conditions – Contract.OldValue**

In the remove method we call the helper Contract.OldValue with the number of animals at the party. What this is doing under the hood, is referring to the value of PartyAnimals.Count in its state before the execution for the method. 

**Assume & Assert**

The Contract.Assert(bool condition) method works as you would expect. It tests some condition, and if finds it to be false then an exception is thrown. Contract.Assume(bool condition) works in the same way, with one additional function – it allows the static checker to add this assumption to the collection of facts it has about your application. 

The animals have tuckered themselves out at this rocking party, and it is time they went home. Thanks for reading, and we will catch you at the next party !
