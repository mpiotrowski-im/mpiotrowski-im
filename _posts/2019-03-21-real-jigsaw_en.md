---
title: Java module system - a real jigsaw
date: 2019-03-21
lang: en
ref: real_jigsaw
summary: The java module system as an element of bigger java ecosystem is currently very immature. I tried to use it in a small project and failed just because one great library which I needed was incompatible with the java module system...
---
# Java module system -- a real jigsaw
I had to write a small application with an user interface. I have decided to use Java 11 and Java FX 11 for the UI. I thought to myself that I could write my first java module application.

I have started with programming an UI prototype. I wrote a *module-info.java* file with all needed declarations: *requires*, *opens*, etc. Everything worked fine. At that moment I was proud how easily I can use the java module system.

Then came time when I needed to add one big open source library. It was the moment when my adventure with java module system had ended. The library which I needed was a long existing one so it did not use the java module system. Like most big libraries it was split into several jar files. Unfortunately the packages with the same names where split into many jar files which ruled out using the library in a module aware application. Compilation returned messages like: `module reads package a.package from both a.impl and a.api`. I had to scrap my idea to learn and use Java module system for the time being...

The library which I needed is currently repackaged so it will be compatible with java module system but the process is still ongoing when I write this post. I can imagine how the repackaging effort could be frustrating...

To summarize: my experience confirmed my opinion that the java module system in its current form is a mistake. Jigsaw project only brought mess into the java ecosystem giving almost nothing in return (my main objection being no support for module versioning). I can imagine how many existing and valuable libraries are incompatible with the java module system for the same reason: splitting API and implementation classes existing in the same packages into several jar files. In my life as a programmer I have seen lots of libraries which did that...

<!--
Copyright 2019 Michał Piotrowski

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0
-->
