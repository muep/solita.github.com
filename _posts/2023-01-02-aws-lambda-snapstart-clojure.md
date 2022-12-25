---
layout: post
title: AWS Lambda SnapStart and Clojure
author: joonas.sarajarvi
excerpt: >
  Shorten the cold startup duration of your AWS Lambda
  functions written in Clojure, by making use of the
  recently added SnapStart feature

tags:
 - AWS
 - Clojure
 - Lambda
 - Performance
---

In November 2022, AWS
[released](https://aws.amazon.com/blogs/aws/new-accelerate-your-lambda-functions-with-lambda-snapstart/)
a new SnapStart feature in their Java 11 runtime for AWS Lambda. As
they explain in their blog entry, enabling SnapStart causes the
function state to be snapshotted right after the `Init` phase of the
function
[lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html#runtimes-lifecycle-ib)
has been completed. Later when Lambda needs to instantiate the
function, it will do that by loading that snapshot instead of starting
from a cold JVM that needs to load up the AWS runtime and the actualy
code provided by the programmer. This can be a substantial improvement
for functions having to load a large number of Java classes, because
getting them from process startup to the end of the `Init` phase may
take several seconds.  Restoring the snapshot, on the other hand,
takes on the order of hundreds of milliseconds.

Cold startup of programs written in [Clojure](https://clojure.org/)
tends to be somewhat slow, and at least
[some](https://ericnormand.me/article/the-legend-of-long-jvm-startup-times)
attribute it exactly to having to load many classes.  Looking at the
description of SnapStart, it seems that it should help with Clojure
programs just as well as with other programs targeting the JVM.  In
case of Clojure, completion of the `Init` phase normally implies
Clojure itself, the handler function and all their dependencies have
been parsed and loaded. So it should be faster, but how much better is
it, actually?

To find out the impact and also just to see if there are some
stumbling blocks, I took some existing Clojure
[code](https://github.com/muep/clj-sysinfo) I had lying around from an
earlier blog
[post](https://dev.solita.fi/2022/03/18/running-clojure-on-constrained-memory.html).
To make it runnable within Lambda, I used this extra namespace:

```clojure
(ns sysinfo.lambda-handler
  (:require [sysinfo :refer [sys-stat]]
            [clojure.data.json :as json])
  (:gen-class
   :methods [[handleStream [java.io.InputStream java.io.OutputStream] void]]))

(defn -handleStream [this in out]
  (spit out (json/write-str {:statusCode 200
                             :body (sys-stat)})))
```

The handler itself is fairly trivial, but it nonetheless `require`s
the earlier program, causing a many classes to be loaded - on top of
the ones coming from just Clojure itself.
