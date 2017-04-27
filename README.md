# Best practices for CouchDB performance on linux systems

## Filesystem
### ext4
#### Mount options (or via /etc/fstab)
`-o errors=remount-ro,noatime,nouser_xattr,barrier=0`

#### Journal
##### Check if exists
`sudo tune2fs -l /dev/sdXY | fgrep has_journal`

##### Turn Off/On
Unmount filesystem (If root filesystem then mount read-only) and then:  
`tune2fs -O ^has_journal /dev/sdXY`

### xfs
#### Mount options (or via /etc/fstab)
`-o noatime,nodiratime,logbufs=8,logbsize=256k,nobarrier`

## /etc/rc.local
```bash
####
## IO Scheduler
####

# First, set an appropriate IO scheduler for file servers:
echo deadline > /sys/block/sdX/queue/scheduler

# Now give the IO scheduler more flexibility by increasing the number of schedulable requests:
echo 4096 > /sys/block/sdX/queue/nr_requests

# To improve throughput for sequential reads, increase the maximum amount of read-ahead data.
# The actual amount of read-ahead is adaptive, so using a high value here won't harm performance for small random access.
echo 4096 > /sys/block/sdX/queue/read_ahead_kb

####
## Virtual memory settings
####

# To avoid long IO stalls (latencies) for write cache flushing in a production environment with very different workloads,
# you will typically want to limit the kernel dirty (write) cache size:
echo 5 > /proc/sys/vm/dirty_background_ratio
echo 10 > /proc/sys/vm/dirty_ratio

# Assigning slightly higher priority to inode caching helps to avoid disk seeks for inode loading:
echo 50 > /proc/sys/vm/vfs_cache_pressure

# Buffering of file system data requires frequent memory allocation.
# Raising the amount of reserved kernel memory will enable faster and more reliable memory allocation in critical situations.
# Raise the corresponding value to 64MB if you have less than 8GB of memory, otherwise raise it to at least 256MB:
echo 262144 > /proc/sys/vm/min_free_kbytes

# For RHEL 7.x and other distributions, it is recommended to have transparent huge pages enabled:
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo always > /sys/kernel/mm/transparent_hugepage/defrag
```
Replace `sdX` with your device name(s).
### Sources:
 - https://www.beegfs.com/wiki/StorageServerTuning
