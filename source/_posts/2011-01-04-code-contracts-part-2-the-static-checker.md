---
title: 'Code Contracts &#8211; Part 2 : The static checker'
author: ammeep
excerpt: 'This  post will cover the options available when enabling static checking of  your project, and a very basic overview of how static checking is  achieved. <br /><br />Code  contracts give us the ability to contractually define invariant  conditions which must be meet during the lifetime of messaging between  two pieces of code. &nbsp;What this means is when Object A calls a method on  Object B, A agrees to ad hear to any contract requirements defined by B.'
layout: post
permalink: /2011/01/04/code-contracts-part-2-the-static-checker/
dsq_thread_id:
  - 952310157
categories:
  - Code
tags:
  - Code
  - Code Contracts
image: http://placekitten.com/200/200
summary: hello and stuff
---
# 

This post will cover the options available when enabling static checking of your project, and a very basic overview of how static checking is achieved.

Code contracts give us the ability to contractually define invariant conditions which must be met during the lifetime of messaging between two pieces of code.  What this means is when Object A calls a method on Object B, A agrees to ad hear to any contract requirements defined by B.

To get compiler assistance for breaches of defined contracts, static checking must be enabled. First ensure that you have installed the code contract tools as per [Part 1][1] of this series. To enable open the properties of your current project, switch to the “Code Contracts” tab and check “Perform Static Contract Checking”. In this section you will see an additional 8 options, each enabling checking of defined contracts.

 [1]: ../../log/2011/1/3/code-contracts-part-1.html

**Implicit Non-Null Obligations**  
This will tell the static checker to look at all the points where messages are being sent between objects and the callers. Warnings will be generated each time the checker is unable to prove mathematically that the caller is NOT passing null values to listeners. By switching on implicit Non-Null Obligations checking, when ever calling code is found as obligated to pass null a recommendation about adding some guard code (in the form of contracts of course) will be made.

To see this in action, let us examine a group of animals getting ready to party

 

        public class Animal
        {
            public string Name { get; private set; }
            public int Age { get; private set; }
     
            public Animal(string name, int age)
            {
                Name = name;
                Age = age;
            }
        }

 

and they all want to attend a party

        public class AnimalParty
        {
            private readonly ICollection _partyAnimals;     
     
            public void AddToAnimalParty(Animal animal)
            {
                _partyAnimals.Add(animal);
            }    
        }

This code looks reasonable and compiles, but there is one major flaw in the PartyAnimals class. If you haven’t been able to spot it, never fear, Code Contracts are here.  By switching on Implicit Non-Null Obligations in the static checking section of the Code Contracts project property, we find out that we are calling a method on a null object because we never assign _partyAnimals.

![][2]  
Assigning _partyAnimals satisfies the inherent obligation (created by calling instance methods on an ICollection) that it must first have an assigned instance.

 [2]: https://lh5.googleusercontent.com/56K8oRlehF5EsiyP0t-PUeRDfn_tTLMqu2qtQXm2Qqiz3j-QjlW6PuVhZbcbOlIo-s_rkDz-MYIiEJd5XURMeXAL7LaRh6I1qL4BhfCvv906HPXJ-g

     private readonly ICollection _partyAnimals;
     
     public AnimalParty()
     {
             _partyAnimals = new List();
      }

The static checker still throws a warning when calling Add() on the party animals collection. We know that this is an instance method, and we have just added initialiser of \_partyAnimals in our constructor. This is a point to note, with no contracts defined that \_partyAnimals can never be null, the static checker is unable to infer that _partyAnimals might never possibly be null. Lets ignore this warning for now and come back to it later.

**Implicit Array Bounds Obligations & Implicit Arithmetic Obligations**  
Checking this option tells the static checker to infer from how an array is used or what arithmetic is performed, what preconditions should be added to your contract to ensure its correctness. This works by deriving proof of inherent assumptions about your variables. When these proofs are violated a warning is generated. This is all achieved by technique called Abstract Interpretation.

**On Abstract Interpretation**  
This is the technique used by the static checker to derive and prove assumptions inherent to your code. Essentially the static checker executes against your program an ‘abstract domain’ which allows the definition of conditions which must hold – or parameters which are constrained within intervals. For example: x is between 0,1 y is between 2, 3 or y = x 2 extrapolate the limit of the function, by increasing the upper bound to infinity. This is only a guess and now needs to be proved.

When the static checker issues a warning about a particular implicit proof obligation or contract, it may not necessarily indicate an error in the code. Warnings are issued whenever the checker cannot prove that the contract holds on all executions generated by the abstract interpretation. Thus encouraging you to evaluate and explicitly declare your assumptions

See this [wikipedia page][3] and this[ video featuring Francesco Logozzo][4] for more information

 [3]: http://en.wikipedia.org/wiki/Abstract_interpretation
 [4]: http://channel9.msdn.com/blogs/peli/static-checking-with-code-contracts-for-net

**Redundant assumptions**  
Enabling this option causes the checker to attempt to prove the Contract.Assume() statements and warn if they are provable. This check is only to see if all your Contract.Assume() are still necessary for the static checker, or can be removed or replaced by asserts. The usage of Contract.Assume() is different when used as part of the static checker or as part of the run time checker. During static verification an assumption is something that will just be added to the facts that are known about the program at that program point, and therefore will aid in the proofs generated by the static checker. At run time Contract.Assume() works like Contract.Assert(), the condition is checked, and if fails then an error is generated.

**And the rest**

Show Squigglies – this option controls the appearance of warnings generated by the static checker in source text. When this option is checked then squigglies appear.

Cahce Results – caches the results of generated proof where the outcome cannot change, thus increasing performance.

Baseline – an interesting option, which allows you to generate a baseline of code where all code at that baseline is deemed ‘proven’ by the static checker and thus will not require a recheck each time you build. This can often be helpful when working with code bases which have not previously used code contracts. This way you can focus your efforts onto warning which are found on new code or changes. To use this option you need to point the baseline XML file. This will be the baseline output directory. After building this baseline file will contain all warnings generated by the static checker. Subsequent runs with the static checker will ignore all warnings which are found in this baseline file.

Next up, we will look further into extending the AnimalParty – defining the rules of engagement with Code Contracts ![:)][5] 

 [5]: http://amy.palamounta.in/wp-content/plugins/php-image-cache/image.php?path=/wp-includes/images/smilies/icon_smile.gif

[![][7]][7]

 []: http://www.dotnetkicks.com/kick/?url=http://amylog.co.nz/log/2011/1/5/code-contracts-part-2-the-static-checker.html