---
title: "Loops, functions and others... a tale of optimization"
date: 2021-04-19T18:10:26+01:00
categories:
- datatent
- perfomance
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
- sharplib.io
- binary search
showTags: true
showPagination: true
showSocial: true
coverImage: https://codemonkeyspace.b-cdn.net/post/2021/gam/featured-image.jpg
coverMeta: in
coverSize: partial
comments: true
thumbnailImage: https://codemonkeyspace.b-cdn.net/post/2021/gam/thumb.webp
thumbnailImagePosition: bottom
showPagination: false
draft: false
math: true
---

This post was supposed to say something about the Global Allocation Map of the Datatent database. But a small function that is not even very significant in this respect turned into a long and interesting optimization problem.

<!--more-->

<!--TOC-->

A short note. When I write it, the allocation is "continuously ascending"; it's not the bit field itself but the byte values. (it gets not filled like 11110000000)

But let's start with the Global Allocation Map here. 

## GAM

GAM is the abbreviation of "Global Allocation Map".

### What is the task of a GAM?

This page aims to track the already allocated pages and hand over the following page id to the management objects.
In standard size, they track $ 65024 (8124 * 8) $ pages. Each of these bits only tells whether the page has already been allocated or not. There is no further information in this page type. A GAM is always the second page after the header in the database file.
And after 65024 pages, always a GAM keeps coming. This pattern makes it easy to get the pages by index in a file.
The first GAM is created with the database file itself and contains one page from the start. The first page in a GAM is always an IAM page, but this comes in a later post.

The allocation of pages is always continuously ascending. This simplification allows many things to be handled more efficiently.
For example, if the last byte is $ 0xFF $, the page is full. When the first byte is $ 0x00 $, the page is empty.
We are searching only for the first set bit when the page is loaded from the disk. After that, the page cache the last issued id.

### How to get a page id

The GAM has a method called AcquirePageId. First, the method search for the following free id. All ids are now relative to the current GAM page and not to the database as a whole. Then the id is marked as allocated. After that, I'm adding the own id of the GAM to the "local" id. Now the id can be returned to the caller and will (hopefully) not returned to the subsequent caller.

##### Finding the last allocated page

It's not hard to find the first not 0xFF byte and then the first not set bit. But it's becoming a challenge for me to make this whole function faster and more optimized. Yes, I know many people will quote this:
{{< blockquote author="Donald Knuth" link="https://en.wikiquote.org/wiki/Donald_Knuth" >}}
The real problem is that programmers have spent far too much time worrying about efficiency in the wrong places and at the wrong times; premature optimization is the root of all evil (or at least most of it) in programming.
{{< /blockquote >}}
But this is a fun and self-education project, so it seems legit to do such things. 
How this works out is part of the next [section](#finding-the-right-value).

##### Mark the allocated page

That is an easy task. Set the bit at the proper position to one.
Because of that, I will only show my current implementation. But maybe there is also a lot of optimization potential in that. :smile:

{{< codecaption lang="csharp" title="mark allocated page" >}}
protected void MarkPageAsAllocated(int localId)
{
    // get the data part of the page
    var dataBuffer = Buffer.Span.Slice(Constants.PAGE_HEADER_SIZE);
    // shortcut for the first byte, DivRem is not needed than
    if (localId < 9)
    {
        ref byte b = ref dataBuffer[0];
        b = (byte)(b | (1 << localId - 1));
        return;
    }

    var bytePos = Math.DivRem(localId, 8, out var remainder);
    if (remainder > 0)
    {
        ref byte b2 = ref dataBuffer[bytePos];
        b2 = (byte)(b2 | (1 << remainder - 1));
    }
    else
    {
        // when remainder == 0, we need to set the last bit of the byte before
        ref byte b2 = ref dataBuffer[bytePos - 1];
        b2 = (byte)(b2 | (1 << 7));
    }
}
{{< /codecaption >}}

## SharpLab.io

I want to present an excellent tool I found when I'm looking around when researching this post. [sharplab.io](https://sharplab.io/) is an online code editor but has very cool features. SharpLib is an online code editor but has very cool features.
It lets you compile with different framework versions. 

You can view different output options (as debug or release build):
- decompiled C# code
- IL code
- assembler after the JIT compiler
- abstract syntax tree
- verfication result of the code
- explains different new C# features in the given code
- result of the code execution

## Finding the correct value

Let's start with the first version.

### Starting Point

The first version has only three real optimizations.
- look at the content as longs, so we need less iterations for searching
- edge cases for empty and full
- use intrinsics

{{< codecaption lang="csharp" title="first version" >}}
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
{{< /codecaption >}}

I use the [BitOperations](https://docs.microsoft.com/en-us/dotnet/api/system.numerics.bitoperations.leadingzerocount?view=net-5.0) class, these is available since .Net Core 3.0.
It uses the processor [intrinsics](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.intrinsics.x86.lzcnt?view=net-5.0) if available and has a optimized software fallback. So I think there wouldn't be any faster method to do this operation.

A good overview over all intrinsics can be found at [Intel](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_lzcnt_u64&expand=3504 "Intel Intrinsics Guide").

###### Results

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
{{</ table >}}

So we can't really optimize the operations.
What can be optimized instead? **The loop**!

### Unrolling

##### What is loop unrolling?

{{< wp tag="Loop_unrolling" lang="en" title="Loop unrolling" >}} (also known as loop unwinding) is a technique to reduce the number of loop executions to gain more performance. 
Like in this small example:

{{< codesidebyside >}}
{{< codeblocksidebyside lang="csharp" pos="0" >}}
int[] arr = new int[100];
for (int i = 0; i < arr.Length; i++)
{
    arr[i] += 5;
}
{{< /codeblocksidebyside >}}
{{< codeblocksidebyside lang="csharp" pos="1" >}}
int[] arr = new int[100];
for (int i = 0; i < arr.Length / 4; i += 4)
{
    arr[i] += 5;
    arr[i + 1] += 5;
    arr[i + 2] += 5;
    arr[i + 3] += 5;
}
{{< /codeblocksidebyside >}}
{{< /codesidebyside >}}

Often this is done by the compiler. But currently, in .Net Core [this is not] always true. It's only appropriate when the content of the loop executes fast and the loop overhead matters. 
It's not a one fits all solution. Often it can be contra-productive and makes the code slower and harder to read. Typically in everyday work, I think it's never helpful to do such an optimization by hand. But here, it's an experiment, so let's look at the code.

This is the code with four times unrolling. I did a test for 2, 4, and 8 times unrolling, but I don't think it's necessary to show all variants.

{{< codecaption lang="csharp" title="four times unrolled" >}}
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
{{< /codecaption >}}

It's not more complicated than the standard loop. I'm only making four comparisons at once. So what does this mean for the performance?
Here the results for different numbers of unrolling.
<br>
<br>

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|          LoopUnrolled4 |       1000 | 102.69 μs | 1.514 μs | 2.171 μs | 102.61 μs |  0.74 |    0.02 |       No |
|          LoopUnrolled2 |       1000 | 155.59 μs | 2.035 μs | 2.918 μs | 156.24 μs |  1.12 |    0.03 |       No |
|          LoopUnrolled8 |       1000 |  77.82 μs | 1.153 μs | 1.690 μs |  77.90 μs |  0.56 |    0.02 |       No |
{{</ table >}}
The first thing we see is that the two times unrolled loop is slower than the unrolled variant. Also, four or eight times unrolling doesn't mean we are four or eight times faster than the standard loop. 

That shows that you should be cautious with manual unrolling, and it needs constantly benchmarking to see if you can make any runtime improvement with it.
But my main question is, why are the two times loop slower than the default one? I'm not an expert in assembly language, but at least with SharpLab.io, it is easy to get the assembler code in the first place.

{{< codeaccordion lang="nasm" title="two times unrolled asm" >}}
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
{{< /codeaccordion >}}

{{< codeaccordion lang="nasm" title="four times unrolled asm" >}}
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
{{< /codeaccordion >}}

As I wrote in the C# code, they are both unrolled, and nothing distinguishes the two in any unique way.
The thing that makes one slower than the normal loop must be something in the processor itself, maybe something like caching or register usage. (I told you, I'm not an expert in this field)

Here we can see that the compiler uses the processor intrinsic to count the number of leading zero bits.

{{< codecaption lang="nasm" >}}
L0150: lzcnt r10, rax
{{< /codecaption >}}

What looks interesting is what does the compiler does here.
I don't understand what's going on here. It makes a left shift by 6, which means 1 shift left by 6 is 64. And it adds a value of 0x38 (56) to a register. Oh, I need to learn more assembler to know what's going on here. 

{{< codesidebyside >}}
{{< codeblocksidebyside lang="csharp" pos="0" hl="2" >}}
if (i > 0 && res != -1)
    res += (64 * i);
{{< /codeblocksidebyside >}}
{{< codeblocksidebyside lang="nasm" pos="1" >}}
L018c: je short L0196
L018e: mov eax, edx
L0190: shl eax, 6
L0193: add r10d, eax
L0196: mov eax, r10d
L0199: add rsp, 0x38
{{< /codeblocksidebyside >}}
{{< /codesidebyside >}}

### Binary Search Like

The next step is to eliminate the loop because we can't improve the loop content much more. So Every iteration, we don't have to do save time.
But how? 

The first thing to notice is because the allocation of pages is stringently increasing, there is never an allocated page after a free one. (currently, of course) This fact means the bit field is like an ordered list. In this case, an algorithm like {{< wp tag="Binary_search_algorithm" lang="en" title="binary search" >}} can be used.

The worst-case performance is $ O(n\log{}n) $ and this is also the average-case performance.
Instead of a maximum of 8124 iterations for the default loop or 2031 for the four times unrolled version, the maximum number of iterations can be calculated by:
$$ log_{2} 8124 \approx 13 $$
So instead of 2031 iterations, it needs 13 to find the value.

<div style="width:50%;margin-left: auto;margin-right: auto;">
{{< figure src="https://codemonkeyspace.b-cdn.net/post/2021/gam/bigo.svg" caption="runtime log vs linear" attr="" attrlink="" width="50%" link="https://codemonkeyspace.b-cdn.net/post/2021/gam/bigo.svg" target="_blank" >}}
</div>

{{< blockquote author="Donald Knuth" link="https://en.wikiquote.org/wiki/Donald_Knuth" >}}
Although the basic idea of binary search is comparatively straightforward, the details can be surprisingly tricky
{{< /blockquote >}}

There are some differences between a standard binary search and the version needed here. It must find the first bit that is not 1, that is in the first byte that isn't 0xFF, or the first long that isn't long.MaxValue.
Because we can search over long, there are only 1016 values to compare, so log(2) 1016 approx 10, fewer comparisons to-do.

And because the first value that matches isn't necessarily the correct one, we need a "lockback" to the value before, if this is also not long.MaxValue the search must go on; otherwise, the correct one is found.

#### Example

{{< figure src="https://codemonkeyspace.b-cdn.net/post/2021/gam/binarysearch.svg" caption="results" attr="" attrlink="" width="100%"link="https://codemonkeyspace.b-cdn.net/post/2021/gam/binarysearch.svg" target="_blank" >}}

##### 1.
Here it's clear that we don't hit the right spot. Our hit is 0 and the value before, so it's not clear that any of these values are correct.
The search must go on.
##### 2.
In this case, the hit also isn't the correct value. But the value before must be the right one. Because it gets allocated in a row, the first value that isn't 0xFF is the right one, and before such a value, it can't be another value as 0xFF. 

##### 3.
Here we hit the right one because the hit isn't 0xFF, and the predecessor is 0xFF.

<br>
<br>
Now the implementation.

##### Code

{{< codecaption lang="csharp" title="binary search like" >}}
public int FindBinarySearch(Span<byte> spanByte)
{
    Span<ulong> span = MemoryMarshal.Cast<byte, ulong>(spanByte);

    if (span[0] == 0)
        return 1;

    if (span[^1] == long.MaxValue)
        return -1;

    int min = 0;
    int max = span.Length - 1;
    int index = -1;

    while (min <= max)
    {
        int mid = mid = (int)unchecked((uint)(min + max) >> 1);
        ref var b = ref span[mid];

        if (b != ulong.MaxValue)
        {
            if (mid == 0)
            {
                index = 0;
                break;
            }

            ref var b1 = ref span[mid - 1];
            if (b1 == ulong.MaxValue)
            {
                index = mid;
                break;
            }

            max = mid - 1;
            continue;
        }

        min = mid + 1;
    }

    if (index > -1)
    {
        int res = 0;
        ref var l = ref span[index];
        var count = BitOperations.LeadingZeroCount((ulong)l);
        res = (64 - count) + 1;
        if (index > 0 && res != -1)
            res += (64 * index);
        return res;
    }

    return index;
}
{{< /codecaption >}}

And now to the results compared with the default loop and the currently fastest implementation.

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|          LoopUnrolled8 |       1000 |  77.82 μs | 1.153 μs | 1.690 μs |  77.90 μs |  0.56 |    0.02 |       No |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
{{</ table >}}

This version beats the eight times unrolled loop clearly, and I think the usage of the binary search algorithm is a more general approach than manual loop unrolling.
But is this the end? Can we make this faster?

### Binary Search Like with Lookup

Now I thought, what can I do now?
My idea was to replace a computation we pre-computed data because that must be faster or not?
I changed this code:


{{< codesidebyside >}}
{{< codeblocksidebyside lang="csharp" pos="0" >}}
int res = 0;
ref var l = ref span[index];
var count = BitOperations.LeadingZeroCount((ulong)l);
res = (64 - count) + 1;
if (index > 0 && res != -1)
    res += (64 * index);
return res;
{{< /codeblocksidebyside >}}
{{< codeblocksidebyside lang="csharp" pos="1" >}}
int res = 0;
ref var l = ref span[index];
res = _lookup[l];
if (index > 0 && res != -1)
    res += (64 * index);
return res;
{{< /codeblocksidebyside >}}
{{< /codesidebyside >}}

This code is the cache builder function:
{{< codecaption lang="csharp" title="cache builder" >}}
static FindFirstEmptyPageBench()
{
    Dictionary<ulong, int> dictionary = new Dictionary<ulong, int>();
    for (int i = 0; i < 64; i++)
    {
        ulong n = 0;
        for (int j = 0; j < i; j++)
        {
            n = (ulong)(n | (1ul << j));
        }

        Debug.WriteLine(i);
        Debug.WriteLine(n);

        if (!dictionary.ContainsKey(n))
            dictionary.Add(i > 0 ? n : ulong.MaxValue, i > 0 ? 64 - i + 1 : 1);
    }
    dictionary.Add(0, 1);
    _lookup = dictionary;
}
{{< /codecaption >}}

And what happens?
{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
| BinarySearchLikeLookup |       1000 |  75.21 μs | 0.746 μs | 1.070 μs |  75.10 μs |  0.54 |    0.01 |       No |
{{</ table >}}

It's slower. That is confusing. Maybe this change stops the compiler from some optimizations. Or is this computation much faster than the lookup from a dictionary? 

So I measured it for 100000 calls.

``` ini
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.19042
AMD Ryzen 7 3700X, 1 CPU, 16 logical and 8 physical cores
.NET Core SDK=6.0.100-preview.2.21155.3
  [Host]    : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT
  MediumRun : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT

Job=MediumRun  IterationCount=15  LaunchCount=2  
WarmupCount=10  
```
{{<table "table table-striped table-sm table-hover tableHeader">}}
|      Method |      Mean |    Error |    StdDev |    Median | Baseline |
|------------ |----------:|---------:|----------:|----------:|--------- |
| Computation |  49.71 μs | 0.449 μs |  0.630 μs |  49.56 μs |       No |
|      Lookup | 570.87 μs | 7.044 μs | 10.102 μs | 568.72 μs |       No |
{{</ table >}}

The computation is much faster than getting the pre-computed value from the dictionary. 
I'm surprised by this result, but this means that precalculation is only worthwhile for much more complex operations. Again, it is of great importance to measure the code to know when this is worthwhile.

### Span.IndexOf

The next try. The built-in functions in the framework are designed for general cases but are already heavily optimized by developers who know what they are doing. So let's try one of the built-in functions.
One thing to try is the [IndexOf](https://docs.microsoft.com/en-us/dotnet/api/system.memoryextensions.indexof?view=net-5.0#System_MemoryExtensions_IndexOf__1_System_Span___0____0_) function on [Span](https://docs.microsoft.com/en-us/dotnet/api/system.span-1?view=net-5.0). 

I recommend looking at the [source code](https://source.dot.net/#System.Private.CoreLib/MemoryExtensions.cs,0d987e9afcdbe004,references) of the framework to see what optimizations are there. 
The test code will be this:

{{< codecaption lang="csharp" title="indexof" >}}
public int IndexOfSearch(Span<byte> spanByte)
{
    Span<ulong> span = MemoryMarshal.Cast<byte, ulong>(spanByte);

    if (span[0] == 0)
        return 1;

    if (span[^1] == long.MaxValue)
        return -1;

    int index = -1;

    var firstZero = span.IndexOf(0u);
    if (firstZero > 0)
    {
        // check prev
        if (span[firstZero - 1] == ulong.MaxValue)
            index = firstZero;
        else
            index = firstZero - 1;
    }
    else
    {
        index = span.Length - 1;
    }

    if (index > -1)
    {
        int res = 0;
        ref var l = ref span[index];
        var count = BitOperations.LeadingZeroCount((ulong)l);
        res = (64 - count) + 1;
        if (index > 0 && res != -1)
            res += (64 * index);
        return res;
    }

    return index;
}
{{< /codecaption >}}

A bit of explanation about my thoughts: I search for the first occurrence of a zero [long]. If something is found, then:
{{< figure src="https://codemonkeyspace.b-cdn.net/post/2021/gam/math.PNG" caption="" attr="" attrlink="" width="100%" link="https://codemonkeyspace.b-cdn.net/post/2021/gam/math.PNG" target="_blank" >}}

otherwise, it must be the last entry [span.Lenght - 1].

How does it perform?

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
|                IndexOf |       1000 | 138.47 μs | 1.586 μs | 2.324 μs | 138.66 μs |  1.00 |    0.03 |       No |
{{</ table >}}
It's on par with the default loop. So no further things to try here. 

### Vector<byte>

Then I asked on [Reddit](https://www.reddit.com/r/csharp/comments/meem45/how_to_optimize_this_function/gsi79qf?utm_source=share&utm_medium=web2x&context=3) if anyone has other ideas for this.
One thing that came up was the use [Vector<T>](https://docs.microsoft.com/de-de/dotnet/api/system.numerics.vector-1?view=net-5.0).

{{< codecaption lang="csharp" title="vector" >}}
public int FindFirstEmptyVectorT(Span<byte> span)
{
    int iterations = Math.DivRem(span.Length, Vector<byte>.Count, out int nonAligned);
    for (int i = 0; i < iterations; i++)
    {
        var vector = new Vector<byte>(span[(Vector<byte>.Count * i)..]);
        if (vector != _testVector)
        {
            int bitIndex = Vector<byte>.Count * i * 8;
            var u64vector = Vector.AsVectorUInt64(vector); // handle LZC with uiint here
            for (int j = 0; j < Vector<ulong>.Count; j++)
            {
                var l = u64vector[j];
                if (l == ulong.MaxValue)
                    continue;

                int res = 0;
                var count = BitOperations.LeadingZeroCount((ulong)l);
                res = (64 - count) + 1;
                if (j > 0 && res != -1)
                    res += (64 * j);
                return res + bitIndex;
            }
        }
    }

    return -1;
}
{{< /codecaption >}}

The results are:

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
|             WithVector |       1000 | 155.86 μs | 2.147 μs | 3.147 μs | 155.26 μs |  1.12 |    0.04 |       No |
{{</ table >}}

It's slower than the standard loop. I believe that this can be made more performant, but I don't will look into it further.


### "Octuple"

The next one from [Reddit](https://www.reddit.com/r/csharp/comments/meem45/how_to_optimize_this_function/gsj33r9?utm_source=share&utm_medium=web2x&context=3) is very special. It uses AVX2 vector intrinsics and does crazy stuff. Please look at the comments in the code for more information.

{{< codecaption lang="csharp" title="indexof" >}}
public unsafe int FindFirstOctupleSearchOnce(Span<byte> spanByte)
{
    Span<uint> span = MemoryMarshal.Cast<byte, uint>(spanByte);
    fixed (uint* spanPtr = span)
    {
        const int bitsPerInt = 32;
        const int ints = 8128 / 4;
        const int loadCount = 8;
        const int sections = loadCount + 1;
        const int sectionLength = (ints / sections) + 1; // 225.8 -> 226
        var indexes = Vector256.Create(
            sectionLength * 1,
            sectionLength * 2,
            sectionLength * 3,
            sectionLength * 4,
            sectionLength * 5,
            sectionLength * 6,
            sectionLength * 7,
            sectionLength * 8);

        int lowerBound = OctupleSearchLowerBound(spanPtr, indexes, sectionLength);
        int index = lowerBound * bitsPerInt;

        int upperBound = Math.Min(lowerBound + sectionLength + 1, span.Length);
        for (int i = lowerBound; i < upperBound; i++)
        {
            int bitsSet = BitOperations.PopCount(span[i]);
            index += bitsSet;
            if (bitsSet != bitsPerInt)
            {
                break;
            }
        }

        return index + 1;
    }
}

public unsafe int OctupleSearchLowerBound(uint* spanPtr, Vector256<int> indexes, int sectionLength)
{
    //Load 8 indexes at once into a Vector256
    var values = Avx2.GatherVector256(spanPtr, indexes, (byte)sizeof(int));

    //How many loaded values have all bits set?
    //If true then set to 0xffffffff else 0
    var isMaxValue = Avx2.CompareEqual(values, Vector256<uint>.AllBitsSet);

    //Take msb of each 32bit element and return them as an int.
    //Then count number of bits that are set and that is equals
    //to the number of loaded values that were all ones.
    var isMaxValueMask = Avx2.MoveMask(isMaxValue.AsSingle());
    var isMaxCount = BitOperations.PopCount((uint)isMaxValueMask);

    //For each loaded vaue that's all ones, a sectionLength
    //number of integers must also be all ones
    return isMaxCount * sectionLength;
}
{{< /codecaption >}}

The results are:

{{<table "table table-striped table-sm table-hover tableHeader">}}
|                 Method | Iterations |      Mean |    Error |   StdDev |    Median | Ratio | RatioSD | Baseline |
|----------------------- |----------- |----------:|---------:|---------:|----------:|------:|--------:|--------- |
|                   Loop |       1000 | 138.85 μs | 1.539 μs | 2.303 μs | 138.74 μs |  1.00 |    0.00 |      Yes |
|       BinarySearchLike |       1000 |  56.73 μs | 1.024 μs | 1.501 μs |  56.48 μs |  0.41 |    0.01 |       No |
|          OctupleSearch |       1000 |  94.76 μs | 1.404 μs | 2.102 μs |  94.30 μs |  0.68 |    0.02 |       No |
{{</ table >}}

It's much better than the standard loop but can't beat the binary search (of course). But it's a clever piece of code and worth looking into. 

## Conclusion

### Is premature optimization truly the root of all evil?

When you need to earn money with your code, this kind of optimization will be, in most cases, contra-productive.
If the user will wait for 0,5s or 0,35s on the result is in most applications irrelevant. Only when you've got code that will be executed many many times than it's helpful to search for hot paths.
But it is always necessary to make sure that you optimize in the right place. Tools like [BenchmarkDotNet](https://benchmarkdotnet.org/) or Profilers like [dotTrace](https://www.jetbrains.com/de-de/profiler/) can be helpful to find the hot paths.

But in a side project, you've got no deadlines, so it's about learning new things, and then it's up to you where to spend your time with.

### And the winner is

<div style="display:flex;justify-content:center;align-items:center;font-weight:bolder;flex-flow: column;">
<div style="width:20%;float:left;">

{{< figure src="https://codemonkeyspace.b-cdn.net/post/2021/gam/podium.svg" caption="" attr="" attrlink="" width="30%" link="https://codemonkeyspace.b-cdn.net/post/2021/gam/podium.svg" target="_blank" >}}

</div>
<span>
Binary search like
</span>
</div>

#### Results

{{< figure src="https://codemonkeyspace.b-cdn.net/post/2021/gam/Find%20first%20zero%20bit.svg" caption="results" attr="" attrlink="" width="100%" link="https://codemonkeyspace.b-cdn.net/post/2021/gam/Find%20first%20zero%20bit.svg" target="_blank" >}}


``` ini
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.19042
AMD Ryzen 7 3700X, 1 CPU, 16 logical and 8 physical cores
.NET Core SDK=5.0.201
  [Host]    : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT
  MediumRun : .NET Core 5.0.4 (CoreCLR 5.0.421.11614, CoreFX 5.0.421.11614), X64 RyuJIT

Job=MediumRun  IterationCount=15  LaunchCount=2  
WarmupCount=10  

```
{{<table "table table-striped table-sm table-hover tableHeader">}}
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
{{</ table >}}

That was my little journey.

<br>
<br>
<br>
<small>
Photo by <a href="https://unsplash.com/@guibolduc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Guillaume Bolduc</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  
 <div>Icons made by <a href="https://www.freepik.com" title="Freepik">Freepik</a> from <a href="https://www.flaticon.com/" title="Flaticon">www.flaticon.com</a></div> 
</small>