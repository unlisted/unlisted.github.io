---
layout: post
title: "mocking subprocess"
date: 2017-01-20
---

I wanted to unit test a function which makes at most three calls to subprocess.Popeni with different arguments. Each Popen call is followed by Popen.communicate() to get the results. 
