---
title: "GAM, Loops and more"
date: 2021-04-09T19:10:26+01:00
categories:
- general
tags:
- datatent
- development
keywords:
- tech
showTags: true
showPagination: true
showSocial: true
coverImage: images/post/2021/gam/featured-image.jpg
coverMeta: in
coverSize: partial
comments: false
thumbnailImage: images/post/2021/gam/thumb.png
thumbnailImagePosition: bottom
showPagination: false
---

This post was supposed to say something about the GAM. But a small function that is not even very significant in 
this respect turned into a long and interesting optimization problem.

<!--more-->

<!--TOC-->

But let's start with the GAM page here. 

## GAM

### What is the task of a GAM?

### How to get a page id

## Finding the right value

### Starting Point

```cs 
public int DefaultLoop(Span<byte> span)
{
    Span<ulong> longSpan = MemoryMarshal.Cast<byte, ulong>(span);

    if (longSpan[0] == 0)
        return 1;

    if (longSpan[^1] == long.MaxValue)
        return -1;

    int iterCount = longSpan.Length / 4;
    for (int i = 0; i < iterCount; i++)
    {
        ref ulong l = ref longSpan[i];
        if (l == ulong.MaxValue)
            continue;
        int res = 0;
        var count = BitOperations.LeadingZeroCount(l);
        res = (64 - count) + 1;
        if (i > 0 && res != -1)
            res += (64 * i);
        return res;
    }

    return -1;
}
```

### Unrolling

```cs 
public int DefaultLoop(Span<byte> span)
{
    Span<ulong> longSpan = MemoryMarshal.Cast<byte, ulong>(span);

    if (longSpan[0] == 0)
        return 1;

    if (longSpan[^1] == long.MaxValue)
        return -1;

    int iterCount = longSpan.Length / 4;
    for (int i = 0; i < iterCount; i++)
    {
        ref ulong l = ref longSpan[i];
        if (l == ulong.MaxValue)
            continue;
        int res = 0;
        var count = BitOperations.LeadingZeroCount(l);
        res = (64 - count) + 1;
        if (i > 0 && res != -1)
            res += (64 * i);
        return res;
    }

    return -1;
}
``` 

### Binary Search Like

### Span.IndexOf

### Vector<byte>

### "Octuple"

## Conclusion


{{< figure src="/images/post/2021/gam/Find first zero bit.svg" caption="results" attr="" attrlink="" width="100%" link="/images/post/2021/gam/Find first zero bit.svg" target="_blank" >}}


``` ini

BenchmarkDotNet=v0.12.1, OS=Windows 10.0.19042
AMD Ryzen 7 3700X, 1 CPU, 16 logical and 8 physical cores
.NET Core SDK=5.0.201
  [Host]    : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT
  MediumRun : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT

Job=MediumRun  IterationCount=15  LaunchCount=2  
WarmupCount=10  

```
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|          LoopUnrolled4 |       1000 | 102.69 μs | 1.514 μs | 2.171 μs | 102.61 μs |  0.74 |    0.02 |       No |
|          LoopUnrolled2 |       1000 | 155.59 μs | 2.035 μs | 2.918 μs | 156.24 μs |  1.12 |    0.03 |       No |
|          LoopUnrolled8 |       1000 |  77.82 μs | 1.153 μs | 1.690 μs |  77.90 μs |  0.56 |    0.02 |       No |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
| BinarySearchLikeLookup |       1000 |  75.21 μs | 0.746 μs | 1.070 μs |  75.10 μs |  0.54 |    0.01 |       No |
|                IndexOf |       1000 | 138.47 μs | 1.586 μs | 2.324 μs | 138.66 μs |  1.00 |    0.03 |       No |
|             WithVector |       1000 | 155.86 μs | 2.147 μs | 3.147 μs | 155.26 μs |  1.12 |    0.04 |       No |
|          OctupleSearch |       1000 |  94.76 μs | 1.404 μs | 2.102 μs |  94.30 μs |  0.68 |    0.02 |       No |



<small>
Photo by <a href="https://unsplash.com/@guibolduc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Guillaume Bolduc</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  </small>