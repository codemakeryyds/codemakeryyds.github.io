---
title: "Tianqi Chen's MLC Course, Chapter 1: What Is Machine Learning Compilation?"
description: "A short overview of the first chapter of the MLC course, focusing on how machine learning compilation turns development-form models into deployable and optimized programs."
date: 2026-05-21T00:00:00+08:00
slug: "mlc-course-overview"
categories:
    - "Course Notes"
tags:
    - "MLC"
    - "Machine Learning Compilation"
    - "TVM"
    - "Deep Learning Systems"
draft: false
---

I recently started reading Tianqi Chen's MLC course. The first chapter mainly answers one question: what does machine learning compilation actually do, and why has it become an important layer in modern machine learning systems?

This note summarizes the core ideas from the introduction. Later notes can go deeper into tensor program abstractions, automatic program optimization, GPU acceleration, and computational graph optimization.

## From Development Form to Deployment Form

When we train or debug a model, we usually work with its development form: model code written in frameworks such as PyTorch, TensorFlow, or JAX, together with the learned parameters. This form is convenient for research, debugging, and fast iteration, but it is not always suitable for direct deployment on phones, browsers, vehicle systems, cloud services, or specialized accelerators.

Production systems need a deployment form. That form includes not only the model computation itself, but also runtime libraries, memory management, hardware calls, application interfaces, and execution support for the target environment.

Machine learning compilation transforms and optimizes a model from its development form into a deployment form that can run efficiently in a specific environment.

## Why Call It Compilation?

A traditional compiler translates high-level programming language code into executable programs. Machine learning compilation has a similar structure: the input is a high-level model description, and the output is a deployable model program, runtime module, or library invocation plan.

However, MLC is not exactly the same as traditional compilation. It does not always generate low-level machine code. Sometimes it rewrites a model into calls to existing high-performance libraries. Sometimes it performs operator fusion, graph optimization, memory optimization, or hardware-specific code generation for GPUs, NPUs, TensorCores, and other accelerators.

In this sense, MLC is more like a system methodology for machine learning deployment. It keeps translating between model representation, computation graphs, tensor programs, and hardware implementations.

## Three Goals of Machine Learning Compilation

The course summarizes the goals of MLC into three categories.

The first goal is integration and dependency minimization. When deploying a model, we only want to include the computation and runtime components that are truly needed. We do not want to ship an entire training framework with the application. This reduces application size and makes deployment easier on resource-constrained devices.

The second goal is hardware acceleration. Different environments provide different hardware capabilities: CPUs, GPUs, mobile NPUs, cloud accelerators, and custom AI chips. MLC maps model computation onto execution patterns that match the target hardware, either by calling native acceleration libraries or by generating low-level programs that use specialized instructions.

The third goal is general optimization. The same model can often be executed in many equivalent ways, but the performance gap between these executions can be large. MLC improves execution through graph optimization, operator fusion, memory reuse, scheduling optimization, and related transformations.

These goals are closely connected. For example, operator fusion can reduce runtime overhead, minimize intermediate memory traffic, and improve hardware utilization at the same time.

## Tensors and Tensor Functions

To understand MLC, it helps to view model execution as computation over tensors.

A tensor is the basic representation for model inputs, outputs, intermediate activations, and weights. Images, text features, hidden states, and weight matrices can all be represented as tensors.

A tensor function describes how tensors are computed from other tensors. A linear layer, a ReLU, and a Softmax can each be viewed as tensor functions. A group of operators, or even an entire model, can also be treated as a larger tensor function.

When MLC optimizes a model, it often looks beyond a single operator. It asks whether a group of tensor functions can be rewritten, fused, or mapped to a more efficient implementation.

![Tensor function transformation in machine learning compilation](image.png)

The figure shows a typical example. In the development form, `linear` and `relu` are two separate computations. In the deployment form, they can be fused into a single `linear_relu` function. This can reduce intermediate tensor reads and writes, while giving the backend more room for optimization.

This is how I understand graph-level operator fusion: it does not change the mathematical meaning of the model. It reorganizes equivalent computation into a form that is more suitable for execution.

## Abstraction and Implementation

One important idea in the first chapter is that abstraction describes what to compute, while implementation describes how to compute it.

In a machine learning system, the same tensor function can appear at many levels. It can be a node in a computation graph, a loop nest, a call to a library function, or a generated GPU kernel.

The core work of MLC is to transform programs across these abstraction layers. High-level abstractions make models easier to express, while low-level implementations bring computation closer to the hardware. The compiler's job is to find a more efficient implementation without changing the program semantics.

This is why machine learning compilation is not just about making a model faster. It connects models, frameworks, runtimes, hardware, and production deployment.

## Why Learn MLC?

For machine learning engineers, MLC helps explain how a model moves from training code to a production environment. When deployment is slow, memory usage is too high, or a target platform is not well supported by mainstream frameworks, changing only the model architecture is often not enough. We also need to understand the execution path underneath.

For algorithm engineers, MLC explains why the same model can have very different performance on different platforms. It also helps us judge whether a model structure is friendly to real-world deployment.

For hardware and systems work, MLC provides a way to connect new hardware capabilities to the machine learning ecosystem. As models grow larger and hardware becomes more diverse, manual adaptation alone becomes difficult to maintain. Compilation and automatic optimization become increasingly important.

## Summary

The first chapter of the MLC course can be summarized in one sentence: machine learning compilation studies how to transform models from development form into deployment form for efficient execution in specific environments.

It is concerned with more than the model itself. It also covers dependency reduction, hardware acceleration, graph optimization, tensor program transformation, and runtime integration. The rest of the course expands these ideas step by step.

Reference: [Machine Learning Compilation, Chapter 1: Introduction](https://book-zh.mlc.ai/chapter_introduction/index.html)
