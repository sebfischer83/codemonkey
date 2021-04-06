---
title: "GAM, Loops and more"
date: 2021-04-09T19:10:26+01:00
categories:
- datatent
tags:
- datatent
- development
- perfomance
keywords:
- database development
- csharp
- c#
- .net
- .net core
- loop unrolling
- perfomance optimization
showTags: true
showPagination: true
showSocial: true
coverImage: images/post/2021/gam/featured-image.jpg
coverMeta: in
coverSize: partial
comments: true
thumbnailImage: images/post/2021/gam/thumb.png
thumbnailImagePosition: bottom
showPagination: false
draft: true
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

```csharp
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

```csharp 
public int Unroll2(Span<byte> span)
{
    Span<ulong> longSpan = MemoryMarshal.Cast<byte, ulong>(span);

    if (longSpan[0] == 0)
        return 1;

    if (longSpan[^1] == long.MaxValue)
        return -1;

    int iterCount = longSpan.Length;
    for (int i = 0; i < iterCount; i += 2)
    {
        ref ulong l2 = ref longSpan[i + 1];
        // when l4 is max value all others before too
        if (l2 == ulong.MaxValue)
            continue;

        ref ulong l1 = ref longSpan[i];

        int res = -1;
        if (l1 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l1);

            res = (64 - count) + 1;
        }
        else if (l2 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l2);
            res = (64) - count + 64 + 1;
        }

        if (i > 0 && res != -1)
            res += (64 * i);

        return res;
    }

    return -1;
}
``` 

```cs
public int Unroll4(Span<byte> span)
{
    Span<ulong> longSpan = MemoryMarshal.Cast<byte, ulong>(span);

    if (longSpan[0] == 0)
        return 1;

    if (longSpan[^1] == long.MaxValue)
        return -1;

    int iterCount = longSpan.Length;
    for (int i = 0; i < iterCount; i += 4)
    {
        ref ulong l4 = ref longSpan[i + 3];
        // when l4 is max value all others before too
        if (l4 == ulong.MaxValue)
            continue;

        ref ulong l1 = ref longSpan[i];
        ref ulong l2 = ref longSpan[i + 1];
        ref ulong l3 = ref longSpan[i + 2];

        int mult = i + 1;
        int res = -1;
        if (l1 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l1);

            res = (64 - count) + 1;
        }
        else if (l2 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l2);
            res = (64) - count + 64 + 1;
        }
        else if (l3 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l3);
            res = (64) - count + 128 + 1;
        }
        else if (l4 != ulong.MaxValue)
        {
            var count = BitOperations.LeadingZeroCount(l4);
            res = (64) - count + 192 + 1;
        }

        if (i > 0 && res != -1)
            res += (64 * i);

        return res;
    }

    return -1;
}
```

```csharp
C.FindFirstEmptyUnroll2(System.Span`1<Byte>)
    L0000: sub rsp, 0x38
    L0004: xor eax, eax
    L0006: mov [rsp+0x28], rax
    L000b: mov [rsp+0x30], rax
    L0010: mov rax, [rdx]
    L0013: mov edx, [rdx+8]
    L0016: shr edx, 3
    L0019: mov [rsp+0x28], rax
    L001e: mov [rsp+0x30], edx
    L0022: cmp dword ptr [rsp+0x30], 0
    L0027: jbe L0124
    L002d: mov rax, [rsp+0x28]
    L0032: cmp qword ptr [rax], 0
    L0036: jne short L0042
    L0038: mov eax, 1
    L003d: add rsp, 0x38
    L0041: ret
    L0042: lea rax, [rsp+0x28]
    L0047: mov edx, [rax+8]
    L004a: lea ecx, [rdx-1]
    L004d: cmp ecx, edx
    L004f: jae L0124
    L0055: mov rax, [rax]
    L0058: movsxd rdx, ecx
    L005b: mov rcx, 0x7fffffffffffffff
    L0065: cmp [rax+rdx*8], rcx
    L0069: jne short L0075
    L006b: mov eax, 0xffffffff
    L0070: add rsp, 0x38
    L0074: ret
    L0075: mov eax, [rsp+0x30]
    L0079: xor edx, edx
    L007b: test eax, eax
    L007d: jle short L00a5
    L007f: lea ecx, [rdx+1]
    L0082: cmp ecx, [rsp+0x30]
    L0086: jae L0124
    L008c: mov r8, [rsp+0x28]
    L0091: movsxd rcx, ecx
    L0094: lea rcx, [r8+rcx*8]
    L0098: cmp qword ptr [rcx], 0xffffffffffffffff
    L009c: jne short L00af
    L009e: add edx, 2
    L00a1: cmp edx, eax
    L00a3: jl short L007f
    L00a5: mov eax, 0xffffffff
    L00aa: add rsp, 0x38
    L00ae: ret
    L00af: cmp edx, [rsp+0x30]
    L00b3: jae short L0124
    L00b5: mov rax, [rsp+0x28]
    L00ba: movsxd r8, edx
    L00bd: lea rax, [rax+r8*8]
    L00c1: mov r8d, 0xffffffff
    L00c7: mov rax, [rax]
    L00ca: cmp rax, 0xffffffffffffffff
    L00ce: je short L00e7
    L00d0: xor r8d, r8d
    L00d3: lzcnt r8, rax
    L00d8: mov ecx, r8d
    L00db: mov r8d, ecx
    L00de: neg r8d
    L00e1: add r8d, 0x41
    L00e5: jmp short L0104
    L00e7: cmp qword ptr [rcx], 0xffffffffffffffff
    L00eb: je short L0104
    L00ed: mov r8, [rcx]
    L00f0: xor eax, eax
    L00f2: lzcnt rax, r8
    L00f7: mov r8d, eax
    L00fa: neg r8d
    L00fd: add r8d, 0x81
    L0104: test edx, edx
    L0106: jle short L0116
    L0108: cmp r8d, 0xffffffff
    L010c: je short L0116
    L010e: mov eax, edx
    L0110: shl eax, 6
    L0113: add r8d, eax
    L0116: mov eax, r8d
    L0119: add rsp, 0x38
    L011d: ret
    L011e: call 0x00007ffc9e07b710
    L0123: int3
    L0124: call 0x00007ffc9e07bc70
    L0129: int3
```

```cs
C.FindFirstEmptyUnroll4(System.Span`1<Byte>)
    L0000: sub rsp, 0x38
    L0004: xor eax, eax
    L0006: mov [rsp+0x28], rax
    L000b: mov [rsp+0x30], rax
    L0010: mov rax, [rdx]
    L0013: mov edx, [rdx+8]
    L0016: shr edx, 3
    L0019: mov [rsp+0x28], rax
    L001e: mov [rsp+0x30], edx
    L0022: cmp dword ptr [rsp+0x30], 0
    L0027: jbe L01a4
    L002d: mov rax, [rsp+0x28]
    L0032: cmp qword ptr [rax], 0
    L0036: jne short L0042
    L0038: mov eax, 1
    L003d: add rsp, 0x38
    L0041: ret
    L0042: lea rax, [rsp+0x28]
    L0047: mov edx, [rax+8]
    L004a: lea ecx, [rdx-1]
    L004d: cmp ecx, edx
    L004f: jae L01a4
    L0055: mov rax, [rax]
    L0058: movsxd rdx, ecx
    L005b: mov rcx, 0x7fffffffffffffff
    L0065: cmp [rax+rdx*8], rcx
    L0069: jne short L0075
    L006b: mov eax, 0xffffffff
    L0070: add rsp, 0x38
    L0074: ret
    L0075: mov eax, [rsp+0x30]
    L0079: xor edx, edx
    L007b: test eax, eax
    L007d: jle short L00a5
    L007f: lea ecx, [rdx+3]
    L0082: cmp ecx, [rsp+0x30]
    L0086: jae L01a4
    L008c: mov r8, [rsp+0x28]
    L0091: movsxd rcx, ecx
    L0094: lea rcx, [r8+rcx*8]
    L0098: cmp qword ptr [rcx], 0xffffffffffffffff
    L009c: jne short L00af
    L009e: add edx, 4
    L00a1: cmp edx, eax
    L00a3: jl short L007f
    L00a5: mov eax, 0xffffffff
    L00aa: add rsp, 0x38
    L00ae: ret
    L00af: cmp edx, [rsp+0x30]
    L00b3: jae L01a4
    L00b9: mov rax, [rsp+0x28]
    L00be: movsxd r8, edx
    L00c1: lea rax, [rax+r8*8]
    L00c5: lea r8d, [rdx+1]
    L00c9: cmp r8d, [rsp+0x30]
    L00ce: jae L01a4
    L00d4: mov r9, [rsp+0x28]
    L00d9: movsxd r8, r8d
    L00dc: lea r8, [r9+r8*8]
    L00e0: lea r9d, [rdx+2]
    L00e4: cmp r9d, [rsp+0x30]
    L00e9: jae L01a4
    L00ef: mov r10, [rsp+0x28]
    L00f4: movsxd r9, r9d
    L00f7: lea r9, [r10+r9*8]
    L00fb: mov r10d, 0xffffffff
    L0101: mov rax, [rax]
    L0104: cmp rax, 0xffffffffffffffff
    L0108: je short L0121
    L010a: xor r10d, r10d
    L010d: lzcnt r10, rax
    L0112: mov r8d, r10d
    L0115: mov r10d, r8d
    L0118: neg r10d
    L011b: add r10d, 0x41
    L011f: jmp short L0184
    L0121: mov rax, [r8]
    L0124: cmp rax, 0xffffffffffffffff
    L0128: je short L0144
    L012a: xor r10d, r10d
    L012d: lzcnt r10, rax
    L0132: mov r9d, r10d
    L0135: mov r10d, r9d
    L0138: neg r10d
    L013b: add r10d, 0x81
    L0142: jmp short L0184
    L0144: mov rax, [r9]
    L0147: cmp rax, 0xffffffffffffffff
    L014b: je short L0167
    L014d: xor r10d, r10d
    L0150: lzcnt r10, rax
    L0155: mov ecx, r10d
    L0158: mov r10d, ecx
    L015b: neg r10d
    L015e: add r10d, 0xc1
    L0165: jmp short L0184
    L0167: cmp qword ptr [rcx], 0xffffffffffffffff
    L016b: je short L0184
    L016d: mov r10, [rcx]
    L0170: xor eax, eax
    L0172: lzcnt rax, r10
    L0177: mov r10d, eax
    L017a: neg r10d
    L017d: add r10d, 0x101
    L0184: test edx, edx
    L0186: jle short L0196
    L0188: cmp r10d, 0xffffffff
    L018c: je short L0196
    L018e: mov eax, edx
    L0190: shl eax, 6
    L0193: add r10d, eax
    L0196: mov eax, r10d
    L0199: add rsp, 0x38
    L019d: ret
    L019e: call 0x00007ffc9e07b710
    L01a3: int3
    L01a4: call 0x00007ffc9e07bc70
    L01a9: int3
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