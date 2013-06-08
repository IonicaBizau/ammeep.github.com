---
title: Ten Traumas In Ten Minutes
author: ammeep
layout: post
permalink: /2012/10/26/ten-traumas-in-ten-minutes/
comments: true
dsq_thread_id:
  - 952222127
categories:
  - Talks
tags:
  - cloud
  - Codemania
  - Deployment
  - talks
image: /images/featured/ten-traumas-ten-minutes.jpg
summary: A 10 minute lightening talk given at Codemania After Dark. In ten minutes I covered my top ten 'traumas' of developing software for the cloud, and along the way gave some thoughts about what you need to keep in mind when doing so. It turns out developing for the cloud is really not that different from developing any other distributed application.
---
# 

Here are the slides from the ten minute lightening talk I gave at [Codemania After Dark][1] this Friday. My very first talk given to an audience of reasonable size – thank you all for coming out.

 [1]: http://codemania.co.nz

The idea behind this talk, was to take you through ten parts of developing for the cloud which can cause headaches or confusion. But as it turns out most of what I mentioned actually applies more at a very broad level – which is awesome because developing for the cloud is really not that different from developing any other application

{% speakerdeck 5084d183df78cd0002012545 %}

The 10 traumas were:

1.  Deployments are painful – Deployment automation in the cloud is easy
2.  I’m married to my cloud –  How to avoid being tied to one cloud
3.  The definition of insanity – Transient fault handling
4.  Limitations of storage – How to work around them
5.  We’ve got to we ‘web scale’ – What does that even mean? You can still use what you already know
6.  Go Daddy goes down  - Simple DNS failover
7.  Our app is slow – Why performance shouldn’t be slapped on at the end
8.  You have lupus – Better diagnostics
9.  The cloud chaos monkey – Be prepared for failures in the cloud
10. Your going to do it wrong – Accept that your designs might change