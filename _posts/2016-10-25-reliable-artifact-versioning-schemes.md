---
title: Reliable artifact versioning schemes
date: 2016-10-25T00:17:37+02:00
excerpt: "Don't derive artifact versions from build numbers"
tags:
  - CI/CD
---

Along with the [Mockito](https://github.com/mockito/mockito) 2.1.0 release the Mockito Team published [an article describing issues with their previous artifact versioning scheme](https://github.com/mockito/mockito/wiki/Continuous-Delivery-Details#why2.1.0).
This reminds me about another versioning scheme that might cause you real trouble, a scheme that is based on CI server's next build number.

Before we get into problems related to this versioning scheme let's quickly discuss some of it's characteristics.
One of the downsides of this scheme is that most of the time it doesn't carry any useful information.
It says nothing about how smooth should the upgrade to the newer version be nor how old the current version is.
But hey, at least build number based scheme guarantees proper ordering of versions, doesn't it?
It also guarantees artifact version uniqueness, doesn't it?
Actually such versioning schemes don't guarantee version uniqueness nor proper ordering.
They don't guarantee that because next build numbers are not under your control.
You have to rely on your CI server assigning them properly.

What happens if you completely loose your CI server?
What if hurricane strikes your data center?
If that seems highly improbable consider a human error, e.g. mistakenly removing some other job than the one you intended to.
That sounds probable, and it can even be automated :D

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">To make error is human. To propagate error to all server in automatic way is <a href="https://twitter.com/hashtag/devops?src=hash">#devops</a>.</p>&mdash; DevOps Borat (@DEVOPS_BORAT) <a href="https://twitter.com/DEVOPS_BORAT/status/41587168870797312">February 26, 2011</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

What if you decide to migrate to another CI server to be able to keep pipelines as a code?
With build number based scheme you have given yourself one more thing to work on.
Now besides migrating your pipelines you also have to migrate next build numbers.
And if you don't migrate build numbers you no longer can rely on the `LATEST` version in your deployment scripts nor on your artifact repository properly selecting the latest artifact.
Moreover due to duplicated artifact versions you will end up being unable to deploy newly built artifacts to artifact repository and thus being unable to deploy to production.

Don't get me wrong, build numbers are not completely wrong, they may sometimes even be useful.
They are just broken when used as a significant part of versioning scheme.
Therefore always make sure you control all significant parts of your versioning scheme or that they are derived from source(s) that guarantees uniqueness and proper ordering.
