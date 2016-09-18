---
layout: single
category: command
author_profile: false
excerpt: ""
title: logsumexp
tags:
comments: true
date: '2016-09-17'
sidebar:
  nav: "commands"
---

[logsumexp] is a convexity aware implementation of \\(\log(\sum_i e^{x_i})\\) 

### Syntax

````matlab
y = logsumexp(x)
````

### Implementation

The convex operator [logsumexp] is implemented using the [evaluation-based nonlinear operators] framework. except when [SCS] or [ECOS]  is used and the exponential cone property is used.

### See also
[entropy], [crossentropy], [kullbackleibler]