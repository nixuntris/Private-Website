---
title: "DAG compression"
description: "About dags"
date: 2023-06-10T04:30:21+01:00
draft: false
slug: "DAG"
image: "dag.jpg"
categories:
    - Compression
tags:
---

Hello, my name is Alex.
I'm building a micro voxel engine called Raine, designed for high-density destructible terrain. One of the biggest hurdles with these kinds of engines? VRAM. And since NVIDIA is still shipping 8 GB consumer cards in 2025, we’re forced to get clever with how we compress voxel data.

What does DAG mean?
Acronym stands for Directed Acyclic Graphs a.k.a DAG
How do they work?
The core idea behind DAG compression is to identify identical subtrees (usually leaf nodes or internal nodes) and store them only once, reusing references elsewhere. This significantly reduces redundancy, especially in procedurally generated or self-similar data.
Why are they so expensive to compute?
You have a lot of data to maul through, and have to basically run a sorting algorithm on all of that, which is hard to parallelize so just porting it to the GPU is not the all be solution and will still need a lot of cycles.
What is a brickmap?
A brickmap stores voxel data in small, fixed-size 3D blocks (“bricks”) instead of one giant array. Each brick—often 8×8×8 or 16×16×16 voxels—can be loaded, updated, or reused independently. This improves memory locality, speeds up streaming, and lets you skip empty or repeated bricks entirely. The trade-off is extra indirection (lookups in a table or tree) and the need to balance brick size: too small wastes metadata space, too large wastes memory on empty air.
How do I store data before compression?
I store them in a static 3D array of uint8_t, where each value refers to a position in a table of a voxel type.

Comparison of Brickmaps, DAGs and a pure 3D array of data on a table.

| Per chunk diagram | Chunk size is 512³ | GPU used for compute: 7900 GRE from AMD |
|-------------------|--------------------|-----------------------------------------|

| Compression technique name | CPU build speed | GPU build speed | Compression rate | Memory before | Memory after |
|----------------------------|-----------------|------------------|------------------|----------------|---------------|
| None                       | None            | None             | 0                | 134 MB         | None          |
| Brickmaps                  | 1s              | 49μs             | ~18x             | ~134 MB        | ~8 MB         |
| DAGs                       | 10s             | 1.5 ms           | ~100x–200x       | ~134 MB        |~0,89(33) MB   |

To note: 
    DAGs compression rate can be much higher in a lot of scenes(where you have less foliage, because it's very irregular and heavy), though this is what I observed on mine. The compression speeds on them also are slow, due to hashing and multiple level deduplication. 

    In one specific scene I observed that the compression rate improvement over brickmaps is non existent. Which were the caves, brickmaps already were easily handling the terrain, and DAGs couldn't really help with folliage(again).

    Every scene is procedurally generated, I don’t use any premade ones.

Potential improvements:
To give more context here, each child is 8x8x8, of course, you can get much better results with DAGs, if you add the ability to have different sizes of them. Though this comes at an expensive computational cost during building. My implementation consists of a static i64 64x64x64 array that references deduplicated children. You can improve on it without changing the technique. This optimization applies regardless of whether you're using brickmaps or DAGs. Here's a table showcasing those results:

|                | Uncompressed | Second Compression Layer | Bit Compression | Both |
|----------------|--------------|---------------------------|------------------|-----|
| compression rate              | ~1           | ~1.3x                         | 2-3x        | ~4-5x       |

Summary
    DAGs are the most space-efficient format I’ve tested


    They’re expensive to compute, even with GPU acceleration


Brickmaps are currently my default in Raine due to build speed, and you can get close to DAG level compression anyways(In foliage heavy scenes).


Future exploration: multi-GPU rendering, why multi-GPU might help with the compression bottlenecks, real-time DAG builds, and custom upscaling pipelines.

Final Thoughts
It's honestly wild that we’re still designing around 8 GB VRAM limits in 2025 — especially in an era of procedural generation and rich physics. I’d love to see how far I could push this with a 7900 XTX or RTX 5090, where memory bandwidth and capacity are far less constrained. Unfortunately, I don’t currently have access to that tier of hardware, but I’m building and testing with what I have.
If you're working on voxel compression, I'd love to hear what structures or tricks you’re using. Hit me up on my Discord server! https://discord.gg/5gz275BwjP
