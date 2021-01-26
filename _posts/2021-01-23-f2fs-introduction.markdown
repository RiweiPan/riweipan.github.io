---
layout: post
title: "[F2FS-Code] <Ch1.Sec1> The Introduction of F2FS"
date: 2021-01-23 16:04:23 +0900
category: F2FS-Code-Scanning
---


# General Introduction
Flash Friendly File System (F2FS) is a log-structured File System (LFS) designed specifically for Flash devices. Compared with the traditional log-structured file system, F2FS has some improvements and optimizations to address the problems such as the high time overhead of the wandering tree and garbage collection. You can refer to the F2FS paper with this [link](https://www.usenix.org/system/files/conference/fast15/fast15-paper-lee.pdf).

> **What is the log-structured File System:**
The most remarkable feature of Log-structured File System (LFS) is to write the new file data to unused blocks, which is known as the out-place-update feature in LFS. This means that even if the old blocks contains data, LFS will still write the new data to unused blocks and invalidate the old blocks. The garbage collection thread then reclaims the old and invalid blocks and resets them to be new/unused blocks for feature use. You can refer to this [link](http://pages.cs.wisc.edu/~remzi/OSTEP/file-lfs.pdf) for more information about LFS.


# F2FS Features

### Storage unit
1. **block:** The minimum storage unint in F2FS is 4KB-size block and the whole storage space is divied into muliple blocks. There are many data structures designed to be 4KB size, which facilitates the interaction with flash devices because falsh devices handles I/Os in multiples of 4KB.
2. **segment:** segment is the structure managing blocks. Each segment contains 512 blocks, so the size of segment is 2MB. Each data allocator (or the log head in LFS) manages a segment, and when all the blocks in the segment are allocated, the data allocator will replace a new segment.
3. **section:** By default, a section manages a segment so that they are one-to-one relationship. Section is the minimim unit for garbage collection (GC). The GC thread will select a victim section to reclaims all blocks in that section.
4. **zone:** By default, one zone = one section = one segment. Zone is associated with the multi-stream flash device, but we do not discuss here.

### The feature of Multi-head Logging in F2FS

The log header area in LFS can be understood as a data allocator so it assigns free blocks for written data and manages the allocation information. In traditonal LFS, they only maintain one log head and all data blocks are assigned from this log head. F2FS considers file's hot-cold access properties and maintains 6 log heads adjusted to different access patterns. This design helps to separate the hot data and cold data so that the garbage collection thread is able to reclaim the appropriate blocks to reduce write amplification. The six log- head is:

- **HOT NODE Area**：Direct node blocks for directories. Because opening and listing dir are most frequent dir operations.
- **WARM NODE Area**：Direct node blocks for regular files.
- **COLD NODE Area**：Indirect node blocks. Indirect node blocks. The blocks are used for the large files.
- **HOT DATA Area**：Directory entry blocks. These blocks records the files and sub-dirs information (file name, dir name) in this dir.
- **WARM DATA Area**：Data blocks made by users. Normal File data.
- **COLD DATA Area**：Data blocks moved by cleaning; Cold data blocks specified by users/Multimedia file data.


# The layout of F2FS

With the help of mkfs.f2fs tool, we can format the whole flash devices into six parts: Superblock, Checkpoint, Segment Info Table, Node Addr Table, Segment Summary Area and Main Area. The first five area is called metadata area, saving the meta information about F2FS (free blocks, block bitmap, etc.). The Main Area stores the dir and file data, including node data, and normal file data. The function of these areas is:

<div align=center>
<img src="/public/img/F2FS-Scanning/F2FS-CH1/f2fs-layout.png" width="950" />
</div>

**Superblock:** Record the meta information about F2FS, including the number of blocks, free blocks, nodes, etc. The corresponding in-memory data structure is `struct f2fs_sb_info`. 

**Checkpoint:** Recording the information about log heads, including the number of free blocks in the log head, and the location where the log head has allocated. F2FS will write the block and node allocation status in Checkpoint Area periodically and F2FS can recover the file system by the information in Checkpoint. The corresponding in-memory data structure is `struct f2fs_checkpoint`. 

**Segment Information Table(SIT):** Record the information of each segment, including the number of free blocks in this segment, and a bitmap of block status. Each segment has a unqiue segment number (segno) and F2FS can acquire the information about segment by segno. The corresponding in-memory data structure is `struct f2fs_sm_info`. 

**Node Address Table(NAT):** Record the information of node. Each node has a node id, which corresponds an entry in NAT. F2FS can obtains storage location of node by referring the NAT with node id.The corresponding in-memory data structure is `struct f2fs_nm_info`. 

**Segment Summary Area(SSA):** Record the data journals and summarys. Journals are used to cache information about temporary changes of SIT and NAT, which can reduce write amplification. Summarys record the relation between logical blocks and physical blocks, and this relation is used in garbage collection operation. There is no in-memory data structure for SSA area.

**Main Area:** Main area is filled with 4KB-size block and these block can store normal file data, dir data and node data.


