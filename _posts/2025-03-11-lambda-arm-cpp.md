---
layout:     post
title:      Cross-compiling C++ to serverless ARM
date:       2025-03-11 08:00
summary:    AWS Lambda meets ARM CPUs
categories: c++
tags:       [c++, serverless, llvm, aws]
giscus_comments: true
---

AWS Lambda is arguably the most popular serverless service.
Lambda functions support natively several programming languages,
such as Python, Node.js, Java or Golang.
However, other languages can be supported through `custom` runtime,
where we deploy a function with a bootstrap script to execute.

Lambda has a unique execution environment for C++. The runtime is based on Amazon Linux 2, which is a CentOS derivative.
In languages like Python or Node.js, shipping the function code to serverless is easy
Since there is no standardized packaging environment in C++,
we 

There are various methods of creating cross-compilation environments,
such as [crostool-ng](https://github.com/crosstool-ng/crosstool-ng).
Alternatively, the entire compilation can be executed within a Docker container
that contains the entire toolchain for a different platform.


no standardized build - so no way to tell if dependency A does not brin ganother one.

First, since we are on x64 Linux and we want to compile Lambda 


First, we need to get the entire toolchain from. To simplify this process, we will use
an existing, containerized cross-compilation toolchain and locate it under the `arm-sysroot` path on our local disk:
We could obtain form the [`dockcross`](dockross) containers.

However, since we need to ship all system libraries as suggested by AWS, and we know the operating environment of Lambda on AWS,
we can instead extract the corss-compilation toolchain from an AWS container.
This will simplify the deployment process since we will be able to skip `libc` and its dependencies, as we know that
this library will be available at the destination.

```bash
mkdir arm-sysroot && docker run --rm dockcross/linux-arm64  tar --dereference -czf - /usr/xcc | tar -xzf - -C arm-sysroot
```


