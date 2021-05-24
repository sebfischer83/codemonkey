---
title: "Hugo shortcodes"
date: 2021-05-24T18:10:26+01:00
categories:
- hugo
tags:
- hugo
keywords:
- hugo
- shortcodes
- code side by side
- code highlighting
- hugo shortcodes
- hugo static site generator
showTags: true
showPagination: true
showSocial: true
coverImage: https://codemonkeyspace.b-cdn.net/post/2021/hugo_shortcodes/featuredimage.webp
coverMeta: in
coverSize: partial
comments: true
thumbnailImage: https://codemonkeyspace.b-cdn.net/post/2021/hugo_shortcodes/featuredimage.webp
thumbnailImagePosition: left
showPagination: false
draft: false
math: true
---

Some Hugo shortcodes that I used in my posts.

<!--more-->

<!--TOC-->

## Code accordion

Collapsable code snippets. Need [Bootstrap5](https://getbootstrap.com/).

### Example

{{< codeaccordion lang="nasm" title="two times unrolled asm" >}}
C.FindFirstEmptyUnroll2(System.Span`1<Byte>)
    L0000: sub rsp, 0x38
    L0004: xor eax, eax
    L0006: mov [rsp+0x28], rax
{{< /codeaccordion >}}

### Code

{{<highlight html>}}
{{/*  https://zwbetz.com/generate-a-random-string-in-hugo/  */}}
{{ $seed := .Get "title" }}
{{ $random := delimit (shuffle (split (md5 $seed) "" )) "" }}

<div class="accordion accordion-flush" id="accordionExample">
    <div class="accordion-item">
        <div class="accordion-header" id="headingOne">
            <button class="accordion-button" type="button" data-bs-toggle="collapse" data-bs-target="#collapseOne{{$random}}" aria-expanded="false" aria-controls="collapseOne">
                <div style="position: absolute;top:0px;font-size:smaller;font-weight:bold;">
                    Code
                </div>
                {{ .Get "title" }}
            </button>
        </div>
          <div id="collapseOne{{$random}}" class="accordion-collapse collapse" aria-labelledby="headingOne" data-bs-parent="#accordionExample">
            <div class="accordion-body">
                <div class="codewrapper">
                    {{ highlight (trim .Inner "\n\r") (.Get "lang") "linenos=false" }}
                  </div>
            </div>
          </div>
    </div>
</div>

{{< /highlight>}}

## Code with caption

### Example

{{< codecaption lang="nasm" title="some title" >}}
L0150: lzcnt r10, rax
{{< /codecaption >}}

### Code

{{<highlight html>}}
<!-- Author: Parsia Hakimian https://github.com/parsiya/Hugo-Shortcodes -->

<!-- codecaption short code - basically this is normal highlighted code with caption on top.
     see readme for usage -->

<figure class="code">
  <figcaption>
      <span>{{ .Get "title" }}</span>
  </figcaption>
  <div class="codewrapper">
    {{ highlight (trim .Inner "\n\r") (.Get "lang") "linenos=false" }}
  </div>
</figure>
{{</highlight>}}

## Code side by side

### Example

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

### Usage

{{<highlight html>}}
{{</* codesidebyside */>}}
{{</* codeblocksidebyside lang="csharp" pos="0" */>}}
int res = 0;
ref var l = ref span[index];
var count = BitOperations.LeadingZeroCount((ulong)l);
res = (64 - count) + 1;
if (index > 0 && res != -1)
    res += (64 * index);
return res;
{{</* /codeblocksidebyside */>}}
{{</* codeblocksidebyside lang="csharp" pos="1" */>}}
int res = 0;
ref var l = ref span[index];
res = _lookup[l];
if (index > 0 && res != -1)
    res += (64 * index);
return res;
{{</* /codeblocksidebyside */>}}
{{</* /codesidebyside */>}}
{{</highlight>}}

### Code

{{<highlight html>}}
{{ $pos := .Get "pos" }}

{{ if eq $pos "0" }}
<div style="flex:1;padding-right:10px;">
{{ else }}
<div style="flex:1;">
{{ end }}
{{ $style := (print "linenos=false,hl_lines=" (.Get "hl") ) }}

{{ highlight (trim .Inner "\n\r") (.Get "lang") $style  }}
</div>
{{</highlight>}}

{{<highlight html>}}
<div style="display:flex;overflow-x:scroll;">
    {{ .Inner }}
</div>
<div style="display: block;clear:both;"></div>
{{</highlight>}}

<br>
<br>
<br>
<small>
Photo by <a href="https://unsplash.com/@der_maik_?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Maik Jonietz</a> on <a href="https://unsplash.com/s/photos/short-code?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</small>