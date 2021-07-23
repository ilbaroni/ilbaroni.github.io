---
layout: post
title:  "Understanding the Apple .DS_Store file format internals to build a parser in Rust"
date:   2021-06-22
categories: research
---



I was trying to find a project to code in Rust, so I thought about understanding the Apple .DS_Store file format and writing a parser in Rust. 

Apple finder creates these files, and they contain records that give attributes to files or directories.

There were situations where I found a .DS_Store file in a web server, and the file itself gave me insights about new files and directories laying around in the root of the webserver.

The records are inside a B-tree, and the pages are all stored in a buddy allocator. The values and offsets in the files are in big-endian.



## Header

The header has 36 bytes:

![ ](/assets/images/apple_dsstore_file_format/1.png)



The first 4-byte integer (**red**) is always `0x00000001` and the magic bytes (**green**) follow, which have the value 0x42756431.

After the magic bytes, there is a sequence of three 4-byte integers. The first and the third (**blue**) represent the offset to the buddy allocator, and both must hold the same value, which is `0x1000`.

The 4-byte integer (**pink**) with the value `0x0800` represents the size of the buddy allocator.



## Buddy allocator

The buddy allocator holds a list of offsets (block addresses section), a free list, and a table of contents (directory section) which maps strings to block numbers.

With the information from the header it is possible to exactly know where the buddy allocator starts and ends. In this case it starts at `0x1000 + 0x4 (padding) = 0x1004` and ends at `0x1004 + 0x800 = 0x1804`.

![ ](/assets/images/apple_dsstore_file_format/1_1.png)



## Block addresses section (offsets)

This section holds a list of offsets to the blocks:

![ ](/assets/images/apple_dsstore_file_format/2.png)



The first 4-byte integer `0x03` (**green**) represents how many offsets will be after the next four null-bytes (**grey**).

In this case, there are tree offsets (**blue**) to be parsed:

````
[0x0000100B, 0x00000045, 0x00000209]
````

>  **Note:** The order of the list is <u>very important</u>, to be able to traverse the data using the block IDs

As seen, the section is padded with enough zeroes (**red**) to round the section up to the next multiple of 256 offsets (1024 bytes). In this example, the next section (directory section) starts at `0x140c`.



## Directory section (table of contents)

This section usually contains at least an entry named `DSDB` with a block ID of `0x01`. 

![ ](/assets/images/apple_dsstore_file_format/3.png)



The bytes in **red** are still padding bytes from the last section. As seen, this section starts with the number of entries to parse (**blue**).

Next, there is a one-byte value (**pink**) that specifies the entry name's length, in this case, `0x04`. The ASCII string (**green**), in this case, `DSDB`, follows.

After the name, there is a 4-byte integer that specifies the block ID (**orange**), which is `0x01`. A parser must use the block ID as an array index to calculate the offset to the block.

This information should be saved in a dictionary:

````
{'DSDB' = 0x01}
````



## B-Tree

The records are all arranged in a B-tree structure. There is in there a master block that contains a pointer to the root node, some statistics, one or more external nodes, and zero or more internal nodes.

The parser will need to traverse the master block to extract information.

### Master block

In the directory section (table of contents), there is at least one entry named `DSDB` with a block ID of `0x01`. In this example, using the block ID as an index `offsets_array[0x1]` gives the value `0x00000045`.

But that value is not "the" offset yet, as it needs to get decoded like the following:

- offset: `0x00000045 >> 0x5 << 0x5` = `0x40 + 0x4 (padding) = 0x44`
- size: `1 << (0x00000045 & 0x1f)` = `0x20`

Now with the decoded offset and the size of the master block, let's inspect the master block:

![ ](/assets/images/apple_dsstore_file_format/4.png)



A master block has five integer values:

- The block number of the root node of the B-tree (**green**)
- The number of internal nodes (**blue**)
- The number of records in the tree (**orange**)
- The number of nodes in the tree, not including the master block (**red**)
- Always `0x1000` (**purple**)

The first integer (**green**) is the most interesting. This value tells the block number of the root node in which the data is in!

In this example, using the block number as an index `offsets_array[0x2]` gives the value `0x00000209`.

Decoded offset and size:

- offset: `0x00000209 >> 0x5 << 0x5` = `0x200 + 0x4 = 0x204`
- size: `1 << (0x00000209 & 0x1f)` = `0x200`

### Root node and the records

![ ](/assets/images/apple_dsstore_file_format/5.png)



The root node starts with the following integers:

* `0x00000000` ; block mode (**blue**)
* `0x0000000A` ; record count (**green**)

If the block is `0x00000000`, then it is followed by `count` records. Otherwise, count pairs with the structure `next_block_id|record` follow. In that case, we would need to parse the record and jump to the next node/block.

In this example, the block mode is `0x00000000`, so there are `0x0A` records to be parsed.

A record starts with the length (**red**) of the `2*length` UTF-16 string that follows (**pink**). After the UTF-16 string, there the structure ID (**purple**) and the structure type (**yellow**).

The length of the data that follows depends on the structure type (**yellow**). [Here](http://search.cpan.org/~wiml/Mac-Finder-DSStore/DSStoreFormat.pod) you can find a list of documented structure types.



## The parser

With this information, I was able to build a parser in Rust (it's a work in progress but already gets the job done). Also, do not judge my Rust skills ^^

[github repo](https://github.com/ilbaroni/dsstore_parser)

Example of the output from the parser:

![ ](/assets/images/apple_dsstore_file_format/6.png)



# References

* [https://0day.work/parsing-the-ds_store-file-format/](https://0day.work/parsing-the-ds_store-file-format/)
* [https://metacpan.org/dist/Mac-Finder-DSStore/view/DSStoreFormat.pod](https://metacpan.org/dist/Mac-Finder-DSStore/view/DSStoreFormat.pod)
* [https://wiki.mozilla.org/DS_Store_File_Format](https://wiki.mozilla.org/DS_Store_File_Format)



