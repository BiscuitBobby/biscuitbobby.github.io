+++
title = "Ramdisk Driver"
date = "2025-01-03T20:19:42+05:30"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "BiscuitBobby"
authorTwitter = "" #do not include @
cover = ""
tags = ["RamDisk", "Linux", "Driver"]
keywords = ["ramdisk", "Block Driver", "ram disk"]
description = "In this post, we'll walk through the a basic RAM disk driver. This driver allocates a chunk of system memory and presents it to the Linux kernel as a standard block device."
showFullContent = false
readingTime = true
hideComments = false
+++

# Building a Simple RAM Disk Block Driver in Linux
**This driver was specifically tested on Linux kernel 6.14.6**

Block devices, like hard drives or USB drives, let you access data anywhere, in fixed-size chunks called 'blocks'. This is different from character devices, which usually handle data sequentially like a stream.

---
In this post, we'll walk through the a basic RAM disk driver. This driver allocates a chunk of system memory and presents it to the Linux kernel as a standard block device. You can then format it with a filesystem (like ext4) and mount it just like any physical drive.x

## Why build this?
It's a great introduction to Linux block driver development, BIO structures, request handling, and memory management within the kernel.

## Diving into the Code
We'll break down the driver piece by piece.

### 1. Includes and Configuration

```c
#include <linux/module.h>    // Core module definitions
#include <linux/kernel.h>    // KERN_INFO, printk, etc.
#include <linux/fs.h>        // File system structures (block_device_operations)
#include <linux/slab.h>      // kmalloc/kfree for small allocations
#include <linux/vmalloc.h>   // vmalloc/vfree for potentially large, virtually contiguous allocations
#include <linux/errno.h>     // Error codes (e.g., -ENOMEM)
#include <linux/hdreg.h>     // (Included, but not used in this simple version)
#include <linux/kdev_t.h>    // MAJOR/MINOR number definitions
#include <linux/blkdev.h>    // Core block device functions (gendisk, BIO, etc.)
#include <linux/bio.h>       // BIO structure definitions
#include <linux/highmem.h>   // kmap/kunmap for accessing high memory pages
#include <linux/spinlock.h>  // Spinlocks for synchronization
```
We need the necessary headers for kernel modules, block devices, memory allocation (`kmalloc` for the device structure, `vmalloc` for the potentially large RAM buffer), BIO handling, and synchronization (`spinlock`).

```c
// --- Configuration ---
#define SRD_DEVICE_NAME "simple_ramdisk"
#define SRD_CAPACITY_MB 16   // Define RAM disk size in MiB
#define SRD_SECTOR_SIZE 512  // Standard sector size
// Calculate capacity in 512-byte sectors
#define SRD_SECTORS (SRD_CAPACITY_MB * 1024 * 1024 / SRD_SECTOR_SIZE)
```

The kernel always works with 'blocks', even if the hardware underneath uses smaller 'sectors'. These kernel blocks have a few rules: their size must be a multiple of the sector size, a power of two, and no larger than a memory page (often 4KB). Common block sizes are 512 bytes, 1KB, or 4KB.

### 2. Module Metadata

```c
MODULE_LICENSE("GPL");
MODULE_AUTHOR("BiscuitBobby");
MODULE_DESCRIPTION("Simple RAM Disk Block Driver");
```

*   Standard kernel module information. `MODULE_LICENSE("GPL")` is crucial for accessing certain kernel symbols.

### 3. The Device Structure
The struct gendisk structure stores information about a disk. This structure is obtained from (In our case) the blk_alloc_disk() call and its fields must be filled before it is sent to the add_disk() function.

```c
// Forward declaration for submit_bio
static void srd_submit_bio(struct bio *bio);

// Device specific structure
struct simple_ramdisk {
    struct gendisk *gd;        // The generic disk structure
    unsigned char *data;       // Pointer to the allocated RAM buffer
    size_t size;               // Size of the RAM buffer in bytes
    spinlock_t lock;           // Lock to protect buffer access
};

// Global storage for our single device instance and major number
static struct simple_ramdisk *srd_dev;
static int srd_major;
```

- **struct simple_ramdisk:** This custom structure bundles everything our driver needs to manage its little slice of the digital world.
    - gd: This points to the gendisk structure – the kernel's official, generic "disk" object. It's how Linux sees our RAM disk.
    - data: The golden ticket! This pointer will hold the address of the actual memory buffer we allocate – our RAM disk's storage space.
    - size: Simply, the size of our data buffer in bytes.
    - lock: A spinlock, our trusty traffic cop, ensuring that access to the data buffer is orderly and prevents digital pile-ups if multiple requests arrive at once.        
- **Global Guides:**
    - srd_dev: A pointer that will hold our one and only RAM disk instance.
    - srd_major: The unique "major number" the kernel assigns our driver, like a postal code for device types.

### 4. Block Device Operations
How does our driver tell the kernel, "Here's how you talk to me"? Through this structure. The kernel uses the file_operations structure to access the driver's functions.

```c
// Block device operations
static const struct block_device_operations srd_ops = {
    .owner = THIS_MODULE,
    // Add .open/.release if needed for more complex state management
    // .open = srd_open,
    // .release = srd_release,
    .submit_bio = srd_submit_bio, // Main I/O handler
};
```

-  **struct block_device_operations**: This structure tells the kernel which functions in our driver handle specific block device actions.
-  **owner**: Set to `THIS_MODULE` to help with reference counting.
- **submit_bio**: For modern block drivers, this function is the primary entry point for all I/O. When the kernel wants to read, write, or do anything else with our RAM disk, it calls our srd_submit_bio function. It's a direct line for handling BIO structures, the fundamental unit of I/O.

### 5. I/O Handling (`srd_submit_bio` and `srd_handle_bio`)

This is where the rubber meets the road – processing those incoming requests from the kernel.

```c
static void srd_handle_bio(struct simple_ramdisk *dev, struct bio *bio)
{
    struct bvec_iter iter;
    sector_t sector_off;
    size_t dev_offset;

    // Check if bio itself is sane first
    if (!bio) {
        pr_err("%s: Received NULL BIO!\n", SRD_DEVICE_NAME);
        return;
    }

    iter = bio->bi_iter; // Get the iterator from the BIO
    sector_off = iter.bi_sector; // Starting sector
    dev_offset = sector_off * SRD_SECTOR_SIZE; // Byte offset in our RAM buffer

    // For DISCARD/WRITE_ZEROES, bi_io_vec might be NULL or segments might be empty.
    // We still need to process the range, but not necessarily map pages.
    if (bio_op(bio) == REQ_OP_DISCARD || bio_op(bio) == REQ_OP_WRITE_ZEROES) {
        // For these operations, we just care about the range given by bio->bi_iter
        size_t total_len_to_process = iter.bi_size; // total size in bytes for the DISCARD/ZERO op

        if (total_len_to_process == 0) { // Nothing to do
            bio->bi_status = BLK_STS_OK;
            return;
        }

        // Check bounds for the entire operation
        if (dev_offset + total_len_to_process > dev->size) {
            pr_err("%s: DISCARD/ZERO Access beyond end of device (sector %llu, offset %zu, len %zu > size %zu)\n",
                   SRD_DEVICE_NAME, (unsigned long long)sector_off, dev_offset, total_len_to_process, dev->size);
            bio->bi_status = BLK_STS_IOERR;
            return;
        }

        spin_lock(&dev->lock);
        memset(dev->data + dev_offset, 0, total_len_to_process);
        spin_unlock(&dev->lock);

        printk(KERN_DEBUG "%s: Discard/Zero %zu bytes at offset %zu\n", SRD_DEVICE_NAME, total_len_to_process, dev_offset);
        bio->bi_status = BLK_STS_OK;
        return; // Handled, no need to iterate bio_vecs
    }

    // For other operations (READ/WRITE), bi_io_vec should be valid.
    // It's still good practice to check bio->bi_io_vec before using it.
    if (!bio->bi_io_vec) {
        pr_err("%s: bio->bi_io_vec is NULL for non-discard/zero op %u, sector %llu\n",
               SRD_DEVICE_NAME, bio_op(bio), (unsigned long long)sector_off);
        bio->bi_status = BLK_STS_IOERR;
        return;
    }

    // Proceed with the loop for READ/WRITE
    do {
        struct bio_vec bvec = bio_iter_iovec(bio, iter); // Should be safe now for R/W
        size_t len = bvec.bv_len;
        unsigned char *ram_addr;
        unsigned char *bio_addr = NULL;

        if (len == 0) { // Skip zero-length segments
            bio_advance_iter_single(bio, &iter, 0);
            continue;
        }

        if (dev_offset + len > dev->size) {
            pr_err("%s: Access beyond end of device (sector %llu, offset %zu, len %zu > size %zu)\n",
                   SRD_DEVICE_NAME, (unsigned long long)sector_off, dev_offset, len, dev->size);
            bio->bi_status = BLK_STS_IOERR;
            break;
        }

        ram_addr = dev->data + dev_offset;

        // bio_op should only be READ or WRITE here due to earlier check
        if (!bvec.bv_page) {
            pr_err("%s: NULL page in BIO for Read/Write op at sector %llu\n",
                   SRD_DEVICE_NAME, (unsigned long long)sector_off);
            bio->bi_status = BLK_STS_IOERR;
            break;
        }
        bio_addr = kmap_local_page(bvec.bv_page) + bvec.bv_offset;

        spin_lock(&dev->lock);
        switch (bio_op(bio)) { // Should only be READ or WRITE
            case REQ_OP_READ:
                memcpy(bio_addr, ram_addr, len);
                printk(KERN_DEBUG "%s: Read %zu bytes at offset %zu\n", SRD_DEVICE_NAME, len, dev_offset);
                break;
            case REQ_OP_WRITE:
                memcpy(ram_addr, bio_addr, len);
                printk(KERN_DEBUG "%s: Write %zu bytes at offset %zu\n", SRD_DEVICE_NAME, len, dev_offset);
                break;
            default:
                // This case should ideally not be reached if the logic above is correct
                pr_warn("%s: Unexpected BIO operation in R/W loop: %d\n", SRD_DEVICE_NAME, bio_op(bio));
                bio->bi_status = BLK_STS_IOERR;
                break;
        }
        spin_unlock(&dev->lock);

        if (bio_addr) {
            kunmap_local(bio_addr);
        }

        if (bio->bi_status != BLK_STS_OK) {
             break;
        }

        dev_offset += len;
        bio_advance_iter_single(bio, &iter, len);

    } while (iter.bi_size > 0);

    return;
}
```

*   **`srd_handle_bio`:**
    The srd_handle_bio function processes I/O requests for the RAM disk.
    
    1. **DISCARD/WRITE_ZEROES operations:**
	    - If the request is to discard or write zeros, the function calculates the total range from the BIO's iterator.
	    - After checking boundaries, it locks access to the RAM disk buffer, uses memset to zero out the specified area, unlocks, sets the BIO status to success, and then finishes. It doesn't loop through individual segments for these operations.
	        
	2. **READ/WRITE operations:**
	    - For reads or writes, it first ensures the BIO's segment array (bio->bi_io_vec) is valid.
	    - Then, it loops through each data segment (struct bio_vec) described in the BIO:
	        - **Bounds Check:** Verifies the segment is within the RAM disk's limits.
	        - **Memory Mapping:** Uses kmap_local_page to make the BIO segment's memory page kernel-accessible.
	        - **Data Transfer:** Acquires a spinlock for safe access to the RAM disk buffer.
	            - For **READs**, data is copied from the RAM disk buffer to the BIO's segment.
	            - For **WRITEs**, data is copied from the BIO's segment to the RAM disk buffer.
	        - The spinlock is released, and the memory page is unmapped using kunmap_local.
	        - If any error occurs during segment processing, an error status is set in the BIO, and the loop may terminate early.
	
	The BIO's status (bio->bi_status) is updated to reflect the outcome of the operation.

```c
// submit_bio callback
static void srd_submit_bio(struct bio *bio) {
    struct simple_ramdisk *dev = bio->bi_bdev->bd_disk->private_data;

    // Handle the actual data transfer or operation
    srd_handle_bio(dev, bio);

    // Signal completion of the BIO
    bio_endio(bio);
}
```
- **srd_submit_bio (The Friendly Dispatcher):**
    - Think of this as the reception desk. It first cleverly finds our simple_ramdisk device data (we tucked it away in private_data earlier).
    - It then hands the bio (the I/O request) off to srd_handle_bio to do the heavy lifting.
    - **Signal completion of BIO request:** After srd_handle_bio finishes its job, bio_endio(bio) is called. This signals back to the kernel (whether it succeeded or failed).

### 6. Device Creation (`create_simple_ramdisk`)

This function sets up all the necessary kernel structures for our block device.

```c
static int create_simple_ramdisk(struct simple_ramdisk **dev_ptr)
{
    struct simple_ramdisk *dev;
    int ret = -ENOMEM; // Assume memory allocation failure initially

    // 1. Allocate our device structure
    dev = kmalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev) {
        printk("%s: Failed to allocate device structure\n", SRD_DEVICE_NAME);
        return -ENOMEM;
    }
    memset(dev, 0, sizeof(*dev));
    spin_lock_init(&dev->lock);

    // 2. Allocate the RAM buffer (using vmalloc for potentially large sizes)
    dev->size = (size_t)SRD_CAPACITY_MB * 1024 * 1024;
    dev->data = vzalloc(dev->size); // vzalloc zeros the memory
    if (!dev->data) {
        printk("%s: Failed to allocate RAM buffer (%zu bytes)\n", SRD_DEVICE_NAME, dev->size);
        kfree(dev);
        return -ENOMEM;
    }
    pr_info("%s: Allocated RAM buffer of %d MiB\n", SRD_DEVICE_NAME, SRD_CAPACITY_MB);

    // 3. Configure Queue Limits
    //    Physical block size often matches logical for simple RAM disks
    struct queue_limits lim = {
        .logical_block_size     = SRD_SECTOR_SIZE,
        .physical_block_size    = SRD_SECTOR_SIZE, // Can be PAGE_SIZE too
        .io_min                 = SRD_SECTOR_SIZE,
        .io_opt                 = PAGE_SIZE, // Optimal I/O is often page size
        .max_sectors            = UINT_MAX, // No real hardware limit
        .max_hw_discard_sectors = UINT_MAX, // Can discard everything
        .max_write_zeroes_sectors = UINT_MAX, // Can write zeroes to everything
    };


    // 4. Allocate Gendisk structure
    //    blk_alloc_disk implicitly creates and sets up the request queue
    dev->gd = blk_alloc_disk(&lim, NUMA_NO_NODE);
    if (IS_ERR(dev->gd)) {
        printk("%s: Failed to allocate gendisk\n", SRD_DEVICE_NAME);
        ret = PTR_ERR(dev->gd);
        goto cleanup_buffer;
    }

    // 5. Initialize Gendisk fields
    dev->gd->major = srd_major;
    dev->gd->first_minor = 0;     // First minor number for this major
    dev->gd->minors = 1;          // Only one device (no partitions)
    dev->gd->fops = &srd_ops;
    dev->gd->private_data = dev;  // Link back to our structure
    snprintf(dev->gd->disk_name, DISK_NAME_LEN, "srd%d", 0); // e.g., srd0

    // 6. Set Capacity
    set_capacity(dev->gd, SRD_SECTORS);
    pr_info("%s: Disk capacity set to %llu sectors (%d MiB)\n",
           SRD_DEVICE_NAME, (unsigned long long)SRD_SECTORS, SRD_CAPACITY_MB);

    // 7. Add Gendisk to System
    ret = add_disk(dev->gd);
    if (ret) {
        printk("%s: Failed to add disk: %d\n", SRD_DEVICE_NAME, ret);
        goto cleanup_disk_obj;
    }

    pr_info("%s: Disk '%s' added successfully\n", SRD_DEVICE_NAME, dev->gd->disk_name);
    *dev_ptr = dev; // Return the successfully created device
    return 0; // Success

// --- Error Handling Cleanup ---
cleanup_disk_obj:
    put_disk(dev->gd); // Release gendisk resources (including queue)
cleanup_buffer:
    vfree(dev->data);
    kfree(dev);
    *dev_ptr = NULL;
    return ret;
}
```

*   **Step 1: Allocate `simple_ramdisk` struct:** Uses `kmalloc` because the structure itself is relatively small.
*   **Step 2: Allocate RAM Buffer:** Uses `vzalloc` (a variant of `vmalloc` that zeroes the memory). `vmalloc` is preferred for large allocations that don't need to be physically contiguous (only virtually contiguous).
*   **Step 3: Queue Limits:** `struct queue_limits` tells the block layer about the device's characteristics (block size, max transfer size, etc.). For a simple RAM disk, basic sector size settings are often sufficient. `blk_alloc_disk` uses these.
*   **Step 4: Allocate Gendisk:** `blk_alloc_disk` allocates the `gendisk` structure *and* implicitly creates an associated request queue. This queue manages incoming BIOs.
*   **Step 5: Initialize Gendisk:** Sets the major/minor numbers, links our `srd_ops`, stores a pointer back to our `dev` structure in `private_data` (crucial for retrieving it in `srd_submit_bio`), and sets the disk name (e.g., "srd0").
*   **Step 6: Set Capacity:** `set_capacity` informs the kernel of the device's size in 512-byte sectors.
*   **Step 7: Add Disk:** `add_disk` registers the `gendisk` with the kernel, making it visible under `/dev/` and usable by the system.
*   **Error Handling:** Uses `goto` for cleanup in case any step fails, ensuring allocated resources are freed in the reverse order of allocation.

### 7. Device Deletion (`delete_simple_ramdisk`)

When it's time to unload our module, we need to be clean it up.

```c
static void delete_simple_ramdisk(struct simple_ramdisk *dev)
{
    if (!dev) return;

    if (dev->gd) {
        del_gendisk(dev->gd); // Remove from system visibility
        put_disk(dev->gd);    // Release gendisk resources (decrements refcount)
    }
    if (dev->data) {
        vfree(dev->data);     // Free the main RAM buffer
    }
    kfree(dev);               // Free our device structure
    pr_info("%s: Device resources released\n", SRD_DEVICE_NAME);
}
```

- Performs cleanup in the reverse order of creation:
    - del_gendisk: Takes the disk offline.
    - put_disk: Releases the gendisk structure and its queue.
    - vfree: Frees our precious RAM buffer.
    - kfree: Frees our simple_ramdisk control structure.

### 8. Module Initialization and Exit (`srd_init`, `srd_exit`)

The main entry and exit points that the kernel calls.

```c
static int __init srd_init(void)
{
    int ret = 0;

    // Register the block device major number
    srd_major = register_blkdev(0, SRD_DEVICE_NAME); // Request dynamic major
    if (srd_major < 0) {
        printk("%s: Failed to register block device: %d\n",
               SRD_DEVICE_NAME, srd_major);
        return srd_major;
    }
    pr_info("%s: Registered with major number %d\n", SRD_DEVICE_NAME, srd_major);

    // Create the actual RAM disk device
    ret = create_simple_ramdisk(&srd_dev);
    if (ret) {
        unregister_blkdev(srd_major, SRD_DEVICE_NAME);
        return ret;
    }

    pr_info("%s: Module loaded successfully\n", SRD_DEVICE_NAME);
    return 0;
}

static void __exit srd_exit(void)
{
    // Delete the block device resources
    delete_simple_ramdisk(srd_dev);

    // Unregister the major number
    if (srd_major > 0) {
        unregister_blkdev(srd_major, SRD_DEVICE_NAME);
        pr_info("%s: Unregistered major number %d\n", SRD_DEVICE_NAME, srd_major);
    }

    pr_info("%s: Module unloaded\n", SRD_DEVICE_NAME);
}

module_init(srd_init);
module_exit(srd_exit);
```

*   **`srd_init`:**
    *   `register_blkdev`: Requests a major number from the kernel. Passing `0` asks for a dynamic (unused) major number, which is the preferred method. Stores the result in `srd_major`.
    *   `create_simple_ramdisk`: Calls our function to allocate memory and set up the `gendisk`.
    *   Handles errors by unregistering the major number if device creation fails.
*   **`srd_exit`:**
    *   `delete_simple_ramdisk`: Calls our cleanup function.
    *   `unregister_blkdev`: Releases the major number back to the kernel.
*   **`module_init`/`module_exit`:** 
	* Macros that specify which functions are the module's initialization and cleanup routines.

### How to Build and Use

1.  **Makefile:** You'll need a simple `Makefile`:
    ```makefile
    obj-m += simple_ramdisk.o
    # Use the name of your C file if different
    simple_ramdisk_driver-objs := simple_ramdisk.o

    KERNEL_SRC ?= /lib/modules/$(shell uname -r)/build

    all:
            $(MAKE) -C $(KERNEL_SRC) M=$(PWD) modules

    clean:
            $(MAKE) -C $(KERNEL_SRC) M=$(PWD) clean
    ```
    *(Save the C code as `simple_ramdisk.c` and the Makefile as `Makefile` in the same directory)*

2.  **Build:** Make sure you have the kernel headers/source installed (usually `linux-headers-$(uname -r)`) and build tools (`build-essential`, `make`). Then run:
    ```bash
    make
    ```
    This will produce `simple_ramdisk.ko`.

3.  **Load:**
    ```bash
    sudo insmod ./simple_ramdisk.ko
    ```
    Check `dmesg` output to see the registered major number and device name (e.g., `srd0`). You should see `/dev/srd0` or a similar output:
	```bash
	[ 5089.573349] simple_ramdisk: Registered with major number 250
	[ 5089.575154] simple_ramdisk: Allocated RAM buffer of 16 MiB
	[ 5089.575199] simple_ramdisk: Disk capacity set to 32768 sectors (16 MiB)
	[ 5089.575503] simple_ramdisk: Read 4096 bytes at offset 0
	[ 5089.575520] simple_ramdisk: Disk 'srd0' added successfully
	[ 5089.575521] simple_ramdisk: Module loaded successfully
	```

4.  **Format:**
    ```bash
    sudo mkfs.ext4 /dev/srd0
    ```
    You'll see the familiar mkfs messages as it lays down the ext4 filesystem structure.

5.  **Mount:**
    ```bash
    sudo mkdir /mnt/ramdisk
    sudo mount /dev/srd0 /mnt/ramdisk
    ```

6.  **Use:** Now you can read/write files in `/mnt/ramdisk`!
	```bash
	root@qemu:/mnt/ramdisk$ ls
	lost+found
	root@qemu:/mnt/ramdisk$ echo "Lemon" >> test.txt
	root@qemu:/mnt/ramdisk$ cat test.txt
	Lemon
	```
7.  **Clean Up (When You're Done):**
    ```bash
    sudo umount /mnt/ramdisk
    sudo rmmod simple_ramdisk
    ```
**(Remember: This is a RAM disk. All its contents vanish into the digital ether when you unload the module or reboot)**

You could extend this by: adding partition support, exploring different queue settings, or even trying to interface with simulated hardware. 
Here is the source code to the driver: [https://github.com/BiscuitBobby/kernel_drivers/blob/main/simple_block/](https://github.com/BiscuitBobby/kernel_drivers/blob/main/simple_block/block_simp.c)

---
This simple RAM disk provides a solid foundation for understanding how Linux interacts with block storage. By walking through its creation you get a practical glimpse into the workings of the kernel's block layer. Happy hacking!
