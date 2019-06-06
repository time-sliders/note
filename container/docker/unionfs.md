# Union mount
## Introduction

In computer operating systems, union mounting is a way of combining multiple directories into one that appears to contain their combined contents. Union mounting is supported in Linux, BSD and several of its successors(继承者，实现类), and Plan 9, with similar but subtly(微妙的) different behavior.
As an example application of union mounting, consider the need to update the information(考虑到更新信息的必要性) contained on a CD-ROM or DVD. While a CD-ROM is not writable, one can overlay(覆盖) the CD's mount point with a writable directory in a union mount. Then, updating files in the union directory will cause them to end up in the writable directory, giving the illusion(错觉) that the CD-ROM's contents have been updated
Ø Implements

# Plan 9

Main article: Plan 9 from Bell Labs § Union directories and namespaces
In the Plan 9 operating system from Bell Labs (mid-1980s onward), union mounting is a central concept(中心概念), replacing several older Unix conventions(协议,习俗) with union directories; for example, several directories containing executables, unioned together at a single /bin directory, replace the PATH variable for command lookup in the shell.

Plan 9 union semantics(语义) are greatly simplified compared to the implementations for POSIX-style operating systems: the union of two directories is simply the concatenation(级联) of their contents, so a directory listing of the union may display duplicate names. Also, no effort is made to recursively merge subdirectories, leading to an extremely simple implementation.Directories are unioned in a controllable order; u/name, where u is a union directory, denotes(表示) the file called name in the first constituent directory that contains such a file.

# Unix and BSD

Unix/POSIX implementations of unions have different requirements(需求) from the Plan 9 implementation due to constraints(约束) in the traditional Unix file system behavior, which greatly complicates(极大复杂了) their implementation and often leads to compromises(妥协).Problems that union mounting on Unix-like operating systems encounters(遭遇到) include:
 
* Duplicate file names within a directory are not acceptable(可接受), since this(因为这) would break applications' expectations(期望) of how a Unix file system works. Putting a logical, stack-like precedence(优先) ordering on the union's constituents(组成部分) partially solves this problem, but requires memory to record which files need to be skipped over during a directory listing (which is otherwise a nearly stateless operation).
* Deletion requires special support: if files with the same name exist in several of the union directory's constituents, simply deleting it from one of the constituents causes a file from one of the others to reappear in its stead.
* Insertion of a directory into the stack can cause incoherency(不一致性) in the kernel's file name cache.
* Renaming a file within a single mounted file system (using the rename system call) should be an atomic operation, but renaming within a union mount can require changes to multiple of the union's constituent directories. A possible solution is to disallow rename in such situations and require implementations to copy-and-delete instead.
* Stable inode numbers for files, hard links and memory-mapped I/O (mmap) are hard to implement correctly. (很难正确实现文件、硬链接和内存映射I/O（MMAP）的稳定Inode Number)
 
Early attempts to add unioning to Unix filesystems included the 3-d filesystem (Bell Labs), the Translucent File Service in SunOS, (Sun Microsystems, 1988[2]) An implementation of union mounting was added to the BSD version of Unix in version 4.4 (1994), taking inspiration from these earlier attempts, Plan 9 and the stackable file systems in Spring (Sun, 1994).[1] 4.4BSD implements the stack-of-directories approach outlined above. Like in Plan 9, operations traverse this stack top-down to resolve names, but unlike in Plan 9, BSD union mounts are recursive, so that the contents of subdirectories appear merged in the union directory. Also unlike in the Plan 9 version, all layers except the top are read-only: modifying files in the union causes their contents to first be copied into the top layer of the stack, where the modifications are then applied. Deletion of files is implemented by writing a special type of file called a whiteout to the top directory, which has the effect of marking the file name as non-existent and hiding files with the same name in the lower layers of the stack. Whiteouts require support from the underlying file system.
 
# Linux

Union mounting was implemented for Linux 0.99 in 1993; this initial implementation was called the Inheriting File System, but was abandoned by its developer because of its complexity. The next major implementation was UnionFS, which grew out of the FiST project at Stony Brook University. An attempt to replace UnionFS, aufs, was released in 2006, followed in 2009 by OverlayFS. Only in 2014 was this last union mount implementation added to the standard Linux kernel source code. 
Similarly, GlusterFS offers a possibility to mount different filesystems distributed across a network, rather than being located on the same machine
 
# UnionFS

## 简介

Unionfs is a filesystem service for Linux, FreeBSD and NetBSD which implements a union mount  for other file systems. It allows files and directories of separate(单独) file systems, known as branches, to be transparently overlaid(透明的覆盖), forming(形成) a single coherent(一致的，连贯的) file system. Contents of directories which have the same path within the merged branches will be seen together in a single merged directory, within the new, virtual filesystem.
 
When mounting branches, the priority of one branch over the other is specified. So when both branches contain a file with the same name, one gets priority over the other.
 
The different branches may be either read-only or read-write file systems, so that writes to the virtual, merged copy are directed to a specific real file system. This allows a file system to appear as writable, but without actually allowing writes to change the file system, also known as copy-on-write. This may be desirable when the media is physically read-only, such as in the case of Live CDs.
(即在只读层上动态添加一层读写层)
 
写入时复制(Copy-on-write)是一个被使用在程序设计领域的最佳化策略。其基础的观念是，如果有多个呼叫者(callers)同时要求相同资源，他们会共同取得相同的指标指向相同的资源，直到某个呼叫者(caller)尝试修改资源时，系统才会真正复制一个副本(private copy)给该呼叫者，以避免被修改的资源被直接察觉到，这过程对其他的呼叫只都是通透的(transparently)。此作法主要的优点是如果呼叫者并没有修改该资源，就不会有副本(private copy)被建立。
。
## Uses

In Knoppix, a union between the file system on the CD-ROM or DVD and a file system contained in an image file called knoppix.img (knoppix-data.img for Knoppix 7) on a writable drive (such as a USB memory stick) can be made, where the writable drive has priority over the read-only filesystem. This allows the user to change any of the files on the system, with the new file stored in the image and transparently used instead of the one on the CD. 
 
Unionfs can also be used to create a single common template for a number of file systems, or for security reasons. It is sometimes used as an ad hoc snapshotting system.
 
Docker uses Unionfs to layer Docker images. As actions are done to a base image, layers are created and documented(新的一层将会被创建并记录), such that each layer fully describes how to recreate an action. This strategy enables Docker's lightweight images, as only layer updates need to be propagated(传播)(compared to full VMs, for example)
 
## Other implementations

Unionfs for Linux has two versions. Version 1.x is a standalone one that can be built as a module. Version 2.x is a newer, redesigned, and reimplemented one.
 
aufs is an alternative(可以替代的) version of unionfs.
AUFS，英文全称是Advanced multi-layered unification(统一) filesystem。AUFS完全重写了早期的UnionFS 1.x，其主要目的是为了可靠性和性能，并且引入了一些新的功能，比如可写分支的负载均衡。AUFS的一些实现已经被纳入UnionFS 2.x版本
 
AUFS是Docker选用的第一种存储驱动。AUFS具有快速启动容器，高效利用存储和内存的优点，直到现在AUFS仍然是Docker支持的一种存储驱动类型。接下来我们要介绍Docker是如何利用AUFS存储images和containers的。
 
 
overlayfs written by Miklos Szeredi has been used in OpenWRT and considered by Ubuntu and has been merged into the mainline Linux kernel on 26 October 2014 after many years of development and discussion for version 3.18 of the kernel.
 
# AUFS

AUFS是一种 Union FS, 简单来说就是“支持将不同目录挂载到同一个虚拟文件系统下的文件系统”, AUFS支持为每一个成员目录设定只读(Rreadonly)、读写(Readwrite)和写(Whiteout-able)权限。Union FS 可以将一个Readonly的Branch和一个Writeable的Branch联合在一起挂载在同一个文件系统下，Live CD正是基于此可以允许在 OS image 不变的基础上允许用户在其上进行一些写操作。Docker在AUFS上构建的Container Image也正是如此。
典型的Linux启动到运行需要两个FileSystem，BootFS 和RootFS。
BootFS 主要包含BootLoader 和Kernel, BootLoader主要是引导加载Kernel, 当Boot成功后，Kernel被加载到内存中BootFS就被Umount了。
RootFS包含的就是典型 Linux 系统中的 /dev、/proc、/bin 等标准目录和文件。
 
不同的linux发行版，BootFS基本是一致的, RootFS会有差别，因此不同的发行版可以共享BootFS。
 
Linux在启动后，首先将RootFS 置为 Readonly，进行一系列检查后将其切换为Readwrite供用户使用。在Docker中，也是利用该技术，然后利用Union Mount在Readonly的RootFS文件系统之上挂载Readwrite文件系统。并且向上叠加, 使得一组Readonly和一个Readwrite的结构构成一个Container的运行目录、每一个被称作一个文件系统Layer。
 
AUFS的特性, 使得每一个对Readonly层文件/目录的修改都只会存在于上层的Writeable层中。这样由于不存在竞争、而且多个Container可以共享Readonly文件系统层。在Docker中，将Readonly的层称作“image” 镜像。对于Container整体而言，整个RootFS变得是read-write的，但事实上所有的修改都写入最上层的writeable层中，image不保存用户状态，可以用于模板、重建和复制。
 
在Docker中，上层的Image依赖下层的Image，因此Docker中把下层的Image称作父Image，没有父Image的Image称作Base Image。因此，想要从一个Image启动一个Container，Docker会先逐次加载其父Image直到Base Image，用户的进程运行在Writeable的文件系统层中。所有父Image中的数据信息以及ID、网络和LXC管理的资源限制、具体container的配置等，构成一个Docker概念上的Container。
 
最后我们总结一下Docker优势，采用AUFS作为Docker的Container的文件系统，能够提供的优势只要有以下几点。
 
多个Container可以共享父Image存储，节省存储空间；快速部署 – 如果要部署多个Container，Base Image可以避免多次拷贝，实现快速部署。因为多个Container共享Image，提高多个Container中的进程命中缓存内容的几率。相比于Copy-on-write类型的FS，Base Image也是可以挂载为可Writeable的，可以通过更新Base Image而一次性更新其之上的Container。