---
layout: post
title:  "On the singleton pattern in C++: Policy-based design"
date: 2017-09-25 19:13:00 -0400
categories: C++
---

[In a previous post]({% post_url 2017-09-24-cpp-singletons %}), we talked about the design of units that use dependency injection to indirectly access singletons, as opposed to unmediated access to global singletons. The singleton implements an interface, and units accept a reference to the interface type. The design is easier to test, maintain, and refactor.