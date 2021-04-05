---
title: "It starts with a page"
subtitle: "423"
date: 2021-03-27T16:10:26+01:00
categories:
- datatent
tags:
- datatent
- development
- page
keywords:
- database
- 
showTags: true
showPagination: true
showSocial: true
coverImage: images/post/2021/it_starts_with_a_page/featured-image.jpg
coverMeta: in
coverSize: partial
comments: false
thumbnailImage: images/post/2021/it_starts_with_a_page/featured-image.jpg
thumbnailImagePosition: right
showPagination: false
disqusIdentifier: cmspage2021
---

This post is about the beginning of the project and the "smallest" part of the database.

<!--more-->

<!-- toc -->

The main target for me is to use only one data file to save all the needed data and meta information. (there will be probably other files for the log)
So, I'm going to use an approach that most databases use to organize the different data types in one file. And that is the **page**.

## What is a page?

The page is the smallest unit of space in Datatent.
A page has a fixed size of **8192 bytes** (but this is configurable at compile time); this means 128 pages per megabyte. Objects can span over multiple pages if they don't fit in one, and a page can contain several objects.
Pages are read and written as the smallest unit to the underlying storage. All I/O operation works on the page level. How this operation will work is part of a later post.
Pages exist in a variety of types, but the underlying principles apply to all.

### Page Header

Each page has a header of **64 bytes** in length. Thirty-two bytes are shared between all pages, and the last 32 bytes are reserved for the use of each page type.

The implementation is a `readonly struct`.

{{< highlight cs >}}
[StructLayout(LayoutKind.Explicit, Size = Constants.PAGE_HEADER_SIZE)] 
internal readonly struct PageHeader
{
    ...
    public static PageHeader FromBuffer(Span<byte> span)
    {
        return MemoryMarshal.Read<PageHeader>(span);
    }

    public void ToBuffer(Span<byte> span)
    {
        Guard.Argument(span.Length).Min(Constants.PAGE_HEADER_SIZE);
        PageHeader a = this;
        MemoryMarshal.Write(span, ref a);
    }
{{< / highlight >}}

Reading and writing are straightforward and fast because it's a `struct`.
With the new [`MemoryMarshal`](https://docs.microsoft.com/de-de/dotnet/api/system.runtime.interopservices.memorymarshal?view=net-5.0)
 class, it's possible to directly write and read a struct from a `Span<byte>`.


The standard header for all page types currently contains the following data.

|Name | Type   | Position  | Length in bytes | Description  |
|---|---|---|---|---|
| Id  | uint  | 0  | 4  | A continuously rising number identifies the page and marks also their position in the file. <br> Database header is always id 0. |
| Type  | byte | 4  | 1  | The type of the page.  |
| Previous id  | uint  | 5  | 4  | The id of the previous page. Not all pages using that field.  |
| Next id  | uint  | 9  | 4  | The id of the next page. Not all pages using that field. |
| Uses bytes | ushort  | 13  | 2  | The number of used bytes stored on the page. Only count raw data and not meta information saved along with the data.  |
| Item count  | byte  | 15  | 1  | The number of items on the page. The maximum number of items is 255 per page.  |
| Next free position  | ushort  | 16 | 2  | Points to the next free byte at the end of all items.  |
| Unaligned free bytes | ushort  | 18  | 2  | The number of free bytes between items.  |
| Highest slot id  | byte  | 20  | 1  | The highest used slot id. |

And the current layout:
```cs
Type layout for 'PageHeader'
Size: 64 bytes. Paddings: 43 bytes (%67 of empty space)
|============================================|
|   0-3: UInt32 PageId (4 bytes)             |
|--------------------------------------------|
|     4: PageType Type (1 byte)              |
| |==============================|           |
| |     0: Byte value__ (1 byte) |           |
| |==============================|           |
|--------------------------------------------|
|   5-8: UInt32 PrevPageId (4 bytes)         |
|--------------------------------------------|
|  9-12: UInt32 NextPageId (4 bytes)         |
|--------------------------------------------|
| 13-14: UInt16 UsedBytes (2 bytes)          |
|--------------------------------------------|
|    15: Byte ItemCount (1 byte)             |
|--------------------------------------------|
| 16-17: UInt16 NextFreePosition (2 bytes)   |
|--------------------------------------------|
| 18-19: UInt16 UnalignedFreeBytes (2 bytes) |
|--------------------------------------------|
|    20: Byte HighestSlotId (1 byte)        |
|--------------------------------------------|
| 21-63: padding (43 bytes)                  |
|============================================|

```

#### Page limits

The maximum number of pages sets the maximum size of the database:
{{< figure src="/images/post/2021/it_starts_with_a_page/math2.png" caption="" attr="" attrlink="" >}}

### Slotted Pages

Slotted pages are pages with slots. What does that mean?
When data from different objects saved to the page, how can we find the data belonging to one entity? One idea is to keep the page id and the offset and length of the data. 
But this approach has one big problem, what happens if we have a lot of fragmentation on the page and need to defrag it? When the entities move, we need to update all references to the data area to update the offset. When we use slots, we only need to update the slot information, but the data's address stays the same. This method simplifies the defragmentation process considerably. Now the entire process can take place within the page.

But there are also page types that don't use slots because their whole space is fixed partitioned.

##### Example

{{< figure src="/images/post/2021/it_starts_with_a_page/page.svg" caption="Data page" attr="" attrlink="" >}}

A slot for an entry contains the data length and the offset on the page. A maximum of 255 slots can be on a page. So a page is filled with 255 entries with approximately 28 bytes in length. I think it will be scarce to have so many small entries, so 255 should be enough by far.

The slots start at the end of the page. That means the slots get filled from the rear. So when there are ten entries, there wouldn't be wasted space for 245 empty slots. The maximum usable length of a data page is calculated by: 
{{< figure src="/images/post/2021/it_starts_with_a_page/math1.png" caption="" attr="" attrlink="" >}}

##### Page Address
```cs 
Type layout for 'PageAddress'
Size: 8 bytes. Paddings: 3 bytes (%37 of empty space)
|================================|
|   0-3: UInt32 PageId (4 bytes) |
|--------------------------------|
|     4: Byte SlotId (1 byte)    |
|--------------------------------|
|   5-7: padding (3 bytes)       |
|================================|
```

A page address is 8 bytes long and consists of the page id and a slot id. There is a reserve of 3 bytes for further usage. 
They are used for all data references outside the page.


### How defragmentation works

This section describes the current simple implementation for defragmentation operation.

{{< figure src="/images/post/2021/it_starts_with_a_page/defrag.svg" caption="defragmentation overview" attr="" attrlink="" >}}

As shown in the picture, the defragmentation operation searches the first and the second gap in the data, when there is no second gap, then until the end and moves the data in the first gap. This procedure is repeated until all gaps are gone, or the maximum number of iterations has been reached.
There are many optimization possibilities here, but this function should typically not be called so often, so this has low priority.
Here is the current source:

```cs
public virtual void Defrag()
{
     // nothing to do
     if (Header.UnalignedFreeBytes == 0)
         return;

     var regions = GetFreeRegions();
     var freeRegions = regions.Count;
     // maximum number of loops to is the initial number of regions with free space
     int maxLoops = regions.Count;

     while (freeRegions > 0 && maxLoops > 0)
     {
         // take the first free region
         var region = regions[0];

         // last free region in the file
         byte nextFreeEntry = HighestDirectoryEntryId;
         if (regions.Count > 1)
         {
             // next free entry starts here, so all between these indexes need to be moved
             var nextRegion = regions[1];
             nextFreeEntry = nextRegion.Item1;
         }

         var between = GetAllDirectoryEntriesBetween(region.Item2, nextFreeEntry);
         var startOffset = between[0].Entry.DataOffset;
         var endOffset = between[^1].Entry.EndPositionOfData();

         var entry = SlotEntry.FromBuffer(Buffer.Span,
             SlotEntry.GetEntryPosition(region.Item1));
         var spanTarget = Buffer.Span.Slice(entry.EndPositionOfData());
         var spanSource = Buffer.Span.Slice(startOffset, endOffset - startOffset);
         // copy all bytes
         spanTarget.WriteBytes(0, spanSource);
         Buffer.Span.Slice(endOffset - region.Item3, region.Item3).Clear();
         for (int i = 0; i < between.Count; i++)
         {
             var entryToChange = SlotEntry.FromBuffer(Buffer.Span, between[i].Index);
             entryToChange = new SlotEntry((ushort) (entryToChange.DataOffset - region.Item3),
                 entryToChange.DataLength);
             entryToChange.ToBuffer(Buffer.Span, SlotEntry.GetEntryPosition(between[i].Index));
         }


         var pageHeader = new PageHeader(Header.PageId, Header.Type, Header.PrevPageId, Header.NextPageId,
             (ushort)(Header.UsedBytes),
             (byte) (Header.ItemCount - 1),
             Header.NextFreePosition,
             (ushort) (regions.Sum(tuple => tuple.Item3) -  region.Item3),
             HighestDirectoryEntryId);
         pageHeader.ToBuffer(Buffer.Span, 0);
         Header = pageHeader;

         regions = GetFreeRegions();
         freeRegions = regions.Count;
         maxLoops -= 1;
     }

     IsDirty = true;
}
```


### Page types

This table shows the planned or existing page types.
  
|Type | Description |
|---|---|
|Header|Contains meta informations about the database. |
|Data|Holds the data.|
|Global Allocation Map (GAM)|Information about the allocated pages.|
|Allocation Information (AIM)|Information what kind of page and the fill factor per page.|
|Attachment|Files that are attached to database objects saved here.|
|Index|Holds index informations.|
|Object Directory (ODM)|A directory of object types and their page addresses.|
|Database Settings|Should save triggers and other changeable things (really unclear currently).|

## How pages are organized

The page organization must be predictable, so it's easy to find the wanted page. My current approach is not the most efficient, but for the beginning, it's enough.

The first three pages are always:
1. Header Page
2. Global Allocation Map Page
3. Allocation Information Page

There are the following rules (with the default settings):
* after the "Global Allocation Map Page" is always an "Allocation Information Page"
* after the "Global Allocation Map Page" there are 65024 pages, then the next GAM follows
* in every GAM there are 64 AIM Pages
* on every AIM Page follows 1015 other pages (an AIM holds 1016 pages but itself as the first one)

{{< figure src="/images/post/2021/it_starts_with_a_page/page_org.svg" caption="data file example" attr="" attrlink="" >}}

Here a short overview of the pages that form the file structure. A detailed overview of each will be in separate posts.

### Global Allocation Map Page

{{< figure src="/images/post/2021/it_starts_with_a_page/gam.svg" caption="GAM" attr="" attrlink="" >}}

This page aims to track which pages are allocated and to retrieve the next available free page for allocation.
It has space to track 65024 pages in a bit field. (8128 bytes * 8 bit per byte) 
For the sake of simplicity, the allocation of pages is always performed in a continuous ascending order.
So it's straightforward to find the following free page id. It's always the first not set bit in the bit field.

### Allocation Information

This page aims to track the type of the page at an id and the fill factor. It is used currently primarily to find a data page with free space to save data there. It tracks the data in a particular struct type. It's 8 bytes long and contains the fill factor, the page type, and the page id.

```csharp
[StructLayout(LayoutKind.Explicit, Size = Constants.ALLOCATION_INFORMATION_ENTRY_SIZE)]
internal readonly struct AllocationInformationEntry
{
    [FieldOffset(PAGE_ID)]
    public readonly uint PageId;

    [FieldOffset(PAGE_TYPE)]
    public readonly PageType PageType;

    [FieldOffset(PAGE_FILL_FACTOR)]
    public readonly PageFillFactor PageFillFactor;

    public AllocationInformationEntry(uint pageId, PageType pageType, PageFillFactor pageFillFactor)
    {
        PageId = pageId;
        PageType = pageType;
        PageFillFactor = pageFillFactor;
    }

    private const int PAGE_ID = 0; // uint 0-3
    private const int PAGE_TYPE = 4; // PageType byte 4
    private const int PAGE_FILL_FACTOR = 5; // PageFillFactor byte 5
}
```

One AIM contains the information for 1016 pages, with the AIM being the first page itself.
Why? This method made it easier to split them into the GAM pages. In this way, one GAM page contains 64 AIM pages. (With the default settings for the page sizes.)


This post was the first short introduction to the project. Although I think there will be many changes, and I will still learn a lot.

<small>
Photo by <a href="https://unsplash.com/@borodinanadi?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">nadi borodina</a> on <a href="/s/photos/page?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</small>