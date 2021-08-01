# File system

[toc]

The purpose of a file system is to organize and store data. File systems typically support sharing of data among users and applications, as well as persistence so that data is still available after a reboot.

## Challenges

1. The file system needs on-disk data structures to represent the tree of named directories and files, to record the identities of the blocks that hold each file’s content, and to record which areas of the disk are free. 
2. The file system must support crash recovery. That is, if a crash (e.g., power failure) occurs, the file system must still work correctly after a restart. The risk is that a crash might interrupt a sequence of updates and leave inconsistent on-disk data structures (e.g., a block that is both used in a file and marked free). 
3. Different processes may operate on the file system at the same time, so the file-system code must coordinate to maintain invariants. 
4. Accessing a disk is orders of magnitude slower than accessing memory, so the file system must maintain an in-memory cache of popular blocks.

## Xv6 file system overview

### Layers

The xv6 file system implementation is organized in seven layers.

![layer](./pic/layers_of_file_system.png)

1. The disk layer reads and writes blocks on an virtio hard drive. 
2. The buffer cache layer caches disk blocks and synchronizes access to them, making sure that only one kernel process at a time can modify the data stored in any particular block. 
3. The logging layer allows higher layers to wrap updates to several blocks in a transaction, and ensures that the blocks are updated atomically in the face of crashes (i.e., all of them are updated or none).
4. The inode layer provides individual files, each represented as an inode with a unique i-number and some blocks holding the file’s data. 
5. The directory layer implements each directory as a special kind of inode whose content is a sequence of directory entries, each of which contains a file’s name and i-number. 
6. The pathname layer provides hierarchical path names like /usr/rtm/xv6/fs.c, and resolves them with recursive lookup. 
7. The file descriptor layer abstracts many Unix resources (e.g., pipes, devices, files, etc.) using the file system interface, simplifying the lives of application programmers.

### Structure

The file system must have a plan for where it stores inodes and content blocks on the disk. To do so, xv6 divides the disk into several sections.

![structure](./pic/structure_of_file_system.png)

- The file system does not use block 0 (it holds the boot sector). 

- Block 1 is called the superblock; it contains:

  - metadata about the file system (the file system size in blocks
  - the number of data blocks
  - the number of inodes
  - the number of blocks in the log

  The superblock is filled in by a separate program, called mkfs, which builds an initial file system.

- Blocks starting at 2 hold the log. 

- After the log are the inodes, with multiple inodes per block. 

- After those come bitmap blocks tracking which data blocks are in use. 

- The remaining blocks are data blocks; each is either marked free in the bitmap block, or holds content for a file or directory. 

## Buffer cache layer

The buffer cache has two jobs: 

1. **synchronize** access to disk blocks to ensure that only one copy of a block is in memory and that only one kernel thread at a time uses that copy.
2. **cache popular blocks** so that they don’t need to be re-read from the slow disk. 

The main interface exported by the buffer cache consists of bread and bwrite

- **bread obtains a buf** containing a copy of a block which can be read or modified in memory
- **bwrite writes a modified buffer to the appropriate block on the disk**. 

A kernel thread must release a buffer by calling **brelse** when it is done with it. The buffer cache uses a per-buffer sleep-lock to ensure that only one thread at a time uses each buffer (and thus each disk block); **bread returns a locked buffer, and brelse releases the lock**.

The buffer cache has a fixed number of buffers to hold disk blocks, which means that if the file system asks for a block that is not already in the cache, the buffer cache must recycle a buffer currently holding some other block. **The buffer cache recycles the least recently used buffer for the new block**. The assumption is that the least recently used buffer is the one least likely to be used again soon.

### Structure of buffer cache

The buffer cache is a doubly-linked list of buffers. The function binit, called by main, initializes the list with the NBUF buffers in the static array buf. All other access to the buffer cache refer to the linked list via bcache.head, not the buf array.

```c++
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

A buffer has two state fields associated with it. 

- The field **valid indicates that the buffer contains a copy of the block**. 
- The field **disk indicates that the buffer content has been handed to the disk**, which may change the buffer (e.g., write data from the disk into data)

```c++
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```

### Code

#### Bread

**Bread calls bget to get a buffer for the given sector**. If the buffer needs to be read from disk, bread calls virtio_disk_rw to do that before returning the buffer. 

```c++
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

Bget scans the buffer list for a buffer with the given device and sector numbers. 

- If there is such a buffer, bget acquires the sleep-lock for the buffer. Bget then **returns the locked buffer.**

- If there is no cached buffer for the given sector, bget must make one, possibly **reusing a buffer** that held a different sector.
  - It scans the buffer list a second time, **looking for a buffer that is not in use** (b->refcnt = 0); any such buffer can be used. 
  - Bget edits the buffer metadata to record the new device and sector number and acquires its sleep-lock. Note that the assignment **b->valid = 0 ensures that bread will read the block data from disk rather than incorrectly using the buffer’s previous contents**.

It is important that **there is at most one cached buffer per disk sector**, to ensure that readers see writes, and because the file system uses locks on buffers for synchronization. 

Bget ensures this invariant by holding the bache.lock continuously from the first loop’s check of whether the block is cached through the second loop’s declaration that the block is now cached (by setting dev, blockno, and refcnt). This causes the check for a block’s presence and (if not present) the designation of a buffer to hold the block to be atomic.

It is safe for bget to acquire the buffer’s sleep-lock outside of the bcache.lock critical section, since the non-zero b->refcnt prevents the buffer from being re-used for a different disk block.

- The sleep-lock protects reads and writes of the block’s buffered content.
- The bcache.lock protects information about which blocks are cached.

#### Bwrite

Once bread has read the disk (if needed) and returned the buffer to its caller, the caller has exclusive use of the buffer and can read or write the data bytes. **If the caller does modify the buffer, it must call bwrite to write the changed data to disk before releasing the buffer**. Bwrite calls virtio_disk_rw to talk to the disk hardware.

#### Brelse

**When the caller is done with a buffer, it must call brelse to release it**. (The name brelse, a shortening of b-release, is cryptic but worth learning: it originated in Unix and is used in BSD, Linux, and Solaris too.) 

Brelse releases the sleep-lock and moves the buffer to the front of the linked list. Moving the buffer causes the list to be ordered by how recently the buffers were used (meaning released): 

- The first buffer in the list is the most recently used
- The last is the least recently used. 

The two loops in bget take advantage of this: 

- The scan for an existing buffer must process the entire list in the worst case, but checking **the most recently used buffers** first (**bcache.head.next**) will reduce scan time when there is good locality of reference. 
- The scan to pick a buffer to reuse picks **the least recently used buffer** by scanning backward (**bcache.head.prev**).

## Logging layer

### Crash recovery

One of the most interesting problems in file system design is **crash recovery**. 

The problem arises because many file-system operations involve multiple writes to the disk, and **a crash after a subset of the writes may leave the on-disk file system in an inconsistent state**. 

For example, suppose a crash occurs during file truncation (setting the length of a file to zero and freeing its content blocks). Depending on the order of the disk writes, the crash may lead to two consequences:

- Either leave an inode with a reference to a content block that is marked free.
- Or leave an allocated but unreferenced content block.

### Solution

Xv6 solves the problem of crashes during file-system operations with a simple form of logging. 

An xv6 system call **does not directly write the on-disk file system data structures**. Instead:

1. It places a description of all the disk writes it wishes to make in a log on the disk. 
2. Once the system call has logged all of its writes, it writes a special commit record to the disk indicating that the log contains a complete operation. 
3. At that point the system call copies the writes to the on-disk file system data structures. 
4. After those writes have completed, the system call erases the log on disk.

If the system should crash and reboot, the file-system code recovers from the crash as follows, **before running any processes**. 

- If the log is marked as containing a complete operation, then the recovery code copies the writes to where they belong in the on-disk file system. 
- If the log is not marked as containing a complete operation, the recovery code ignores the log. The recovery code finishes by erasing the log.

Why does xv6’s log solve the problem of crashes during file system operations? 

- If the crash occurs before the operation commits, then the log on disk will not be marked as complete, the recovery code will ignore it, and the state of the disk will be as if the operation had not even started. 

- If the crash occurs after the operation commits, then recovery will replay all of the operation’s writes, perhaps repeating them if the operation had started to write them to the on-disk data structure. 

In either case, **the log makes operations atomic with respect to crashes: after recovery, either all of the operation’s writes appear on the disk, or none of them appear**.

### Log design

**The log resides at a known fixed location, specified in the superblock**. 

It consists of **a header block followed by a sequence of updated block copies** (“logged blocks”). 

- The header block contains an array of sector numbers, one for each of the logged blocks, and the count of log blocks. The count in the header block on disk is either zero, indicating that there is no transaction in the log, or non-zero, indicating that the log contains a complete committed transaction with the indicated number of logged blocks. 
- Xv6 writes the header block when a transaction commits, but not before, and sets the count to zero after copying the logged blocks to the file system. Thus a crash midway through a transaction will result in a count of zero in the log’s header block; a crash after a commit will result in a non-zero count.

Each system call’s code indicates the start and end of the sequence of writes that must be atomic with respect to crashes. To allow concurrent execution of file-system operations by different processes, **the logging system can accumulate the writes of multiple system calls into one transaction**. Thus **a single commit may involve the writes of multiple complete system calls**. To avoid splitting a system call across transactions, **the logging system only commits when no file-system system calls are underway**.

The idea of committing several transactions together is known as **group commit**. 

- Group commit reduces the number of disk operations because it amortizes the fixed cost of a commit over multiple operations. 
- Group commit also hands the disk system more concurrent writes at the same time, perhaps allowing the disk to write them all during a single disk rotation. 

Xv6 dedicates a fixed amount of space on the disk to hold the log. **The total number of blocks written by the system calls in a transaction must fit in that space**. This has two consequences:

1. No single system call can be allowed to write more distinct blocks than there is space in the log. This is not a problem for most system calls, but two of them can potentially write many blocks: write and unlink. 

   - A large file write may write many data blocks and many bitmap blocks as well as an inode block.
   - Unlinking a large file might write many bitmap blocks and an inode. 

   Xv6’s write system call breaks up large writes into multiple smaller writes that fit in the log, and unlink doesn’t cause problems because in practice the xv6 file system uses only one bitmap block. 

2. The other consequence of limited log space is that the logging system cannot allow a system call to start unless it is certain that the system call’s writes will fit in the space remaining in the log.

### Code

A typical use of the log in a system call looks like this:

```c++
begin_op();
...
bp = bread(...);
bp->data[...] = ...;
log_write(bp);
...
end_op();
```

#### begin_op

**begin_op waits until the logging system is not currently committing, and until there is enough unreserved log space to hold the writes from this call.** 

- log.outstanding counts the number of system calls that have reserved log space; the total reserved space is log.outstanding times MAXOPBLOCKS. 
- Incrementing log.outstanding both reserves space and prevents a commit from occuring during this system call. 
- The code conservatively assumes that each system call might write up to MAXOPBLOCKS distinct blocks.

```c++
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

#### log_write

**log_write acts as a proxy for bwrite**. 

- It records the block’s sector number in memory

- reserving it a slot in the log on disk

- pins the buffer in the block cache to prevent the block cache from evicting it.

  why pins the buffer?

  The block must stay in the cache until committed: 

  - until then, the cached copy is the only record of the modification
  - it cannot be written to its place on disk until after commit
  - other reads in the same transaction must see the modifications. 

```c++
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

log_write notices when **a block is written multiple times during a single transaction, and allocates that block the same slot in the log**. This optimization is often called **absorption**. 

Why needs absorption?

It is common that, for example, the disk block containing inodes of several files is written several times within a transaction. By absorbing several disk writes into one, the file system can **save log space** and can **achieve better performance** because only one copy of the disk block must be written to disk.

#### end_op

end_op first decrements the count of outstanding system calls. If the count is now zero, it commits the current transaction by calling commit(). There are four stages in this process:

1. write_log() copies each block modified in the transaction from the buffer cache to its slot in the log on disk. 

2. **write_head() writes the header block to disk**: **this is the commit point, and a crash after the write will result in recovery replaying the transaction’s writes from the log.** 

3. install_trans reads each block from the log and writes it to the proper place in the file system. 

4. Finally end_op writes the log header with a count of zero.

   this has to happen before the next transaction starts writing logged blocks, so that a crash doesn’t result in recovery using one transaction’s header with the subsequent transaction’s logged blocks.

```c++
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

#### recover_from_log

recover_from_log is called from initlog, which is called from fsinit during boot before the first user process runs. It reads the log header, and mimics the actions of end_op if the header indicates that the log contains a committed transaction.

```c++
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```

## Inode layer

The term inode can have one of two related meanings (structure name or entity). 

- It might refer to the on-disk data structure containing a file’s size and list of data block numbers. 
- Or “inode” might refer to an in-memory inode, which contains a copy of the on-disk inode as well as extra information needed within the kernel.

### Structure

#### On-disk inodes

The on-disk inodes are packed into a **contiguous area of disk** called the inode blocks. Every inode is the same size, so it is easy, given a number n, to find the nth inode on the disk. In fact, this number n, called the inode number or i-number, is how inodes are identified in the implementation.

The on-disk inode is defined by a struct dinode. 

```c++
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

- The **type** field distinguishes between files, directories, and special files (devices). A type of zero indicates that an on disk inode is free. 
- The **nlink** field counts the number of directory entries that refer to this inode, in order to recognize when the on-disk inode and its data blocks should be freed. 
- The **size** field records the number of bytes of content in the file. 
- The **addrs** array records the block numbers of the disk blocks holding the file’s content.

The inode data is found in the blocks listed in the dinode ’s addrs array. 

- The first NDIRECT blocks of data are listed in the first NDIRECT entries in the array; these blocks are called direct blocks. 
- The next NINDIRECT blocks of data are listed not in the inode but in a data block called the indirect block. The last entry in the addrs array gives the address of the indirect block. 

Thus the first 12 kB ( NDIRECT x BSIZE) bytes of a file can be loaded from blocks listed in the inode, while the next 256 kB ( NINDIRECT x BSIZE) bytes can only be loaded after consulting the indirect block. 

![dinode](./pic/dinode.png)

#### In-memory inodes

```c++
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

The kernel keeps the set of active inodes in memory; **struct inode is the in-memory copy of a struct dinode on disk**. 

- The kernel stores an inode in memory only if there are C pointers referring to that inode. 
- The ref field counts the number of C pointers referring to the in-memory inode, and the kernel discards the inode from memory if the reference count drops to zero. 
- The iget and iput functions acquire and release pointers to an inode, modifying the reference count. 

Pointers to an inode can come from file descriptors, current working directories, and transient kernel code such as exec.

#### lock-like mechanism

There are four lock or lock-like mechanisms in xv6’s inode code. 

1. **icache.lock protects the invariant that an inode is present in the cache at most once**, and the invariant that a cached inode’s ref field counts the number of in-memory pointers to the cached inode. 
2. Each in-memory inode has a **lock** field containing a sleep-lock, which ensures exclusive access to the inode’s fields (such as file length) as well as to the inode’s file or directory content blocks. 
3. An inode’s **ref**, if it is greater than zero, causes the system to maintain the inode in the cache, and not re-use the cache entry for a different inode. 
4. Finally, each inode contains a **nlink** field (on disk and copied in memory if it is cached) that counts the number of directory entries that refer to a file; xv6 won’t free an inode if its link count is greater than zero.

#### Inode cache

The **inode cache** only caches inodes to which kernel code or data structures hold C pointers. Its **main job** is really **synchronizing access by multiple processes**; **caching is secondary**. 

If an inode is used frequently, the buffer cache will probably keep it in memory if it isn’t kept by the inode cache. The inode cache is write-through, which means that code that modifies a cached inode must immediately write it to disk with iupdate.

### Code

#### ialloc

Ialloc is similar to balloc: 

1. It loops over the inode structures on the disk, one block at a time, looking for one that is marked free. 
2. When it finds one, it claims it by writing the new type to the disk and then returns an entry from the inode cache with the tail call to iget. 

```c++
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode.
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb));
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      return iget(dev, inum);
    }
    brelse(bp);
  }
  panic("ialloc: no inodes");
}
```

The correct operation of ialloc depends on the fact that **only one process at a time can be holding a reference to bp**: ialloc can be sure that some other process does not simultaneously see that the inode is available and try to claim it.

#### iget

**A struct inode pointer returned by iget() is guaranteed to be valid until the corresponding call to iput().**

- The inode won’t be deleted, and the memory referred to by the pointer won’t be re-used for a different inode. 
- iget() provides non-exclusive access to an inode, so that there can be many pointers to the same inode. 

Many parts of the file-system code depend on this behavior of iget(), both to hold long-term references to inodes (as open files and current directories) and to prevent races while avoiding deadlock in code that manipulates multiple inodes (such as pathname lookup).

Acquisition of inode pointers(iget) and locking inode(ilock) are separated.

- The struct inode that iget returns may not have any useful content. **In order to ensure it holds a copy of the on-disk inode, code must call ilock**. 
- "ilock" locks the inode (so that no other process can ilock it) and reads the inode from the disk, if it has not already been read. iunlock releases the lock on the inode. 

**Separating acquisition of inode pointers from locking helps avoid deadlock in some situations**, for example during directory lookup. Multiple processes can hold a C pointer to an inode returned by iget, but only one process can lock the inode at a time.

```c++
// Find the inode with number inum on device dev
// and return the in-memory copy. Does not lock
// the inode and does not read it from disk.
static struct inode*
iget(uint dev, uint inum)
{
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // Is the inode already cached?
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    if(empty == 0 && ip->ref == 0)    // Remember empty slot.
      empty = ip;
  }

  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");

  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```

Iget looks through the inode cache for an active entry (ip->ref > 0) with the desired device and inode number. If it finds one, it returns a new reference to that inode. 

As iget scans, it records the position of the first empty slot, which it uses if it needs to allocate a cache entry.

#### ilock and iunlock

Code must lock the inode using ilock before reading or writing its metadata or content. Ilock uses a sleep-lock for this purpose. Once ilock has exclusive access to the inode, it reads the inode from disk (more likely, the buffer cache) if needed. The function iunlock releases the sleep-lock, which may cause any processes sleeping to be woken up.

```c++
// Lock the given inode.
// Reads the inode from disk if necessary.
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");

  acquiresleep(&ip->lock);

  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

```c++
// Unlock the given inode.
void
iunlock(struct inode *ip)
{
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");

  releasesleep(&ip->lock);
}
```

#### iput

Iput releases a C pointer to an inode by **decrementing the reference count**. If this is the last reference, the inode’s slot in the inode cache is now free and can be re-used for a different inode. 

If iput sees that there are no C pointer references to an inode and that the inode has no links to it (occurs in no directory), then the inode and its data blocks must be freed. 

- Iput calls itrunc to truncate the file to zero bytes, freeing the data blocks
- sets the inode type to 0 (unallocated)
- writes the inode to disk. 

```c++
// Drop a reference to an in-memory inode.
// If that was the last reference, the inode cache entry can
// be recycled.
// If that was the last reference and the inode has no links
// to it, free the inode (and its content) on disk.
// All calls to iput() must be inside a transaction in
// case it has to free the inode.
void
iput(struct inode *ip)
{
  acquire(&icache.lock);

  if(ip->ref == 1 && ip->valid && ip->nlink == 0){
    // inode has no links and no other references: truncate and free.

    // ip->ref == 1 means no other process can have ip locked,
    // so this acquiresleep() won't block (or deadlock).
    acquiresleep(&ip->lock);

    release(&icache.lock);

    itrunc(ip);
    ip->type = 0;
    iupdate(ip);
    ip->valid = 0;

    releasesleep(&ip->lock);

    acquire(&icache.lock);
  }

  ip->ref--;
  release(&icache.lock);
}
```

The locking protocol in iput in the case in which it frees the inode deserves a closer look. 

- One danger is that a concurrent thread might be waiting in ilock to use this inode (e.g., to read a file or list a directory), and won’t be prepared to find that the inode is not longer allocated. 

  This can’t happen because there is no way for a system call to get a pointer to a cached inode if it has no links to it and ip->ref is one. That one reference is the reference owned by the thread calling iput. It’s true that iput checks that the reference count is one outside of its icache.lock critical section, but at that point the link count is known to be zero, so no thread will try to acquire a new reference. 

- The other main danger is that a concurrent call to ialloc might choose the same inode that iput is freeing. 

  This can only happen after the iupdate writes the disk so that the inode has type zero. This race is benign; the allocating thread will politely wait to acquire the inode’s sleep-lock before reading or writing the inode, at which point iput is done with it. 

iput() can write to the disk. **This means that any system call that uses the file system may write the disk,** because the system call may be the last one having a reference to the file. Even calls like read() that appear to be read-only, may end up calling iput(). This, in turn, means that **even read-only system calls must be wrapped in transactions if they use the file system**.

**There is a challenging interaction between iput() and crashes**. 

- iput() doesn’t truncate a file immediately when the link count for the file drops to zero, because some process might still hold a reference to the inode in memory: a process might still be reading and writing to the file, because it successfully opened it. 
- But, if a crash happens before the last process closes the file descriptor for the file, then the file will be marked allocated on disk but no directory entry will point to it.

File systems handle this case in one of two ways. 

1. The simple solution is that on recovery, after reboot, the file system scans the whole file system for files that are marked allocated, but have no directory entry pointing to them. If any such file exists, then it can free those files.
2. The second solution doesn’t require scanning the file system. In this solution, the file system records on disk (e.g., in the super block) the inode inumber of a file whose link count drops to zero but whose reference count isn’t zero. If the file system removes the file when its reference counts reaches 0, then it updates the on-disk list by removing that inode from the list. On recovery, the file system frees any file in the list.

**Xv6 implements neither solution, which means that inodes may be marked allocated on disk, even though they are not in use anymore**. This means that over time xv6 runs the risk that it may run out of disk space.

#### bmap

 The function bmap manages the representation so that higher-level routines such as readi and writei, which we will see shortly. Bmap returns the disk block number of the bn’th data block for the inode ip. If ip does not have such a block yet, bmap allocates one.

```c++
// Inode content
//
// The content (data) associated with each inode is stored
// in blocks on the disk. The first NDIRECT block numbers
// are listed in ip->addrs[].  The next NINDIRECT blocks are
// listed in block ip->addrs[NDIRECT].

// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

The function bmap begins by picking off the easy case: the first NDIRECT blocks are listed in the inode itself. The next NINDIRECT blocks are listed in the indirect block at ip->addrs[NDIRECT]. Bmap reads the indirect block and then reads a block number from the right position within the block. If the block number exceeds NDIRECT+NINDIRECT, bmap panics; writei contains the check that prevents this from happening.

Bmap allocates blocks as needed. An ip->addrs[] or indirect entry of zero indicates that no block is allocated. As bmap encounters zeros, it replaces them with the numbers of fresh blocks, allocated on demand.

#### itrunc

itrunc frees a file’s blocks, resetting the inode’s size to zero. 

Itrunc starts by freeing the direct blocks, then the ones listed in the indirect block, and finally the indirect block itself.

```c++
// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

#### readi

Readi starts by making sure that the offset and count are not beyond the end of the file. Reads that start beyond the end of the file return an error while reads that start at or cross the end of the file return fewer bytes than requested. The main loop processes each block of the file, copying data from the buffer into dst.

```c++
// Read data from inode.
// Caller must hold ip->lock.
// If user_dst==1, then dst is a user virtual address;
// otherwise, dst is a kernel address.
int
readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return 0;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyout(user_dst, dst, bp->data + (off % BSIZE), m) == -1) {
      brelse(bp);
      tot = -1;
      break;
    }
    brelse(bp);
  }
  return tot;
}
```

#### writei

writei is identical to readi, with three exceptions: 

1. writes that start at or cross the end of the file grow the file, up to the maximum file size.
2. the loop copies data into the buffers instead of out
3. if the write has extended the file, writei must update its size.

```c++
// Write data to inode.
// Caller must hold ip->lock.
// If user_src==1, then src is a user virtual address;
// otherwise, src is a kernel address.
// Returns the number of bytes successfully written.
// If the return value is less than the requested n,
// there was an error of some kind.
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      break;
    }
    log_write(bp);
    brelse(bp);
  }

  if(off > ip->size)
    ip->size = off;

  // write the i-node back to disk even if the size didn't change
  // because the loop above might have called bmap() and added a new
  // block to ip->addrs[].
  iupdate(ip);

  return tot;
}
```

#### stati

The function stati copies inode metadata into the stat structure, which is exposed to user programs via the stat system call.

```c++
// Copy stat information from inode.
// Caller must hold ip->lock.
void
stati(struct inode *ip, struct stat *st)
{
  st->dev = ip->dev;
  st->ino = ip->inum;
  st->type = ip->type;
  st->nlink = ip->nlink;
  st->size = ip->size;
}
```

