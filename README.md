# Linux tuning for better CouchDB performance

## Filesystem tuning
### ext4
#### Mount options (or via /etc/fstab)
`errors=remount-ro,noatime,nouser_xattr,barrier=0`

#### Journal
(Replace `sdXY` with your partition name)
##### Check if exists
`sudo tune2fs -l /dev/sdXY | fgrep has_journal`

##### Turn Off/On
Unmount filesystem (If root filesystem then mount read-only) and then:  
`tune2fs -O ^has_journal /dev/sdXY`

### xfs
#### Mount options (or via /etc/fstab)
`noatime,nodiratime,logbufs=8,logbsize=256k,nobarrier`

## /etc/rc.local
(Replace `sdX` with your device name)
```bash
####
## IO Scheduler
####

# First, set an appropriate IO scheduler for file servers:
echo deadline > /sys/block/sdX/queue/scheduler

# Now give the IO scheduler more flexibility by increasing the number of schedulable requests:
echo 4096 > /sys/block/sdX/queue/nr_requests

# To improve throughput for sequential reads, increase the maximum amount of read-ahead data.
# The actual amount of read-ahead is adaptive,
# so using a high value here won't harm performance for small random access.
echo 4096 > /sys/block/sdX/queue/read_ahead_kb

####
## Virtual memory settings
####

# To avoid long IO stalls (latencies) for write cache flushing
# in a production environment with very different workloads,
# you will typically want to limit the kernel dirty (write) cache size:
echo 5 > /proc/sys/vm/dirty_background_ratio
echo 10 > /proc/sys/vm/dirty_ratio

# Assigning slightly higher priority to inode caching helps
# to avoid disk seeks for inode loading:
echo 50 > /proc/sys/vm/vfs_cache_pressure

# Decrease swappiness to prevent swapping as much as possible
echo 1 > /proc/sys/vm/swappiness

# Buffering of file system data requires frequent memory allocation.
# Raising the amount of reserved kernel memory will enable faster and more reliable
# memory allocation in critical situations.
# Raise the corresponding value to 64MB if you have less than 8GB of memory,
# otherwise raise it to at least 256MB:
echo 262144 > /proc/sys/vm/min_free_kbytes

# For RHEL 7.x and other distributions, it is recommended to have transparent huge pages enabled:
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo always > /sys/kernel/mm/transparent_hugepage/defrag

####
## Process scheduler
####

# There's a kernel parameter that determines how long a migrated process has to be running
# before the kernel will consider migrating it again to another core.
# The sysctl name is sched_migration_cost_ns, default value 50000 (that's ns so 0.5 ms).
# Forking servers, like PostgreSQL or Apache, scale to much higher levels of concurrent
# connections if this is made larger, by at least an order of magnitude:
echo 5000000 > /proc/sys/kernel/sched_migration_cost_ns

# Another parameter that can dramatically impact forking servers is sched_autogroup_enabled.
# This setting groups tasks by TTY, to improve perceived responsiveness on an interactive system.
# On a server with a long running forking daemon, this will tend to keep child processes from
# migrating away as soon as they should.
# It can be disabled like so:
echo 0 > /proc/sys/kernel/sched_autogroup_enabled
```
### Apply the changes
`sudo /etc/rc.local` or `reboot`

## ionice
Giving CouchDB IO priority with `ionice`: `sudo ionice -p $(pidof beam.smp) -c 1 -n 0`.  
[This can also be done in a `systemd` unit](https://gist.github.com/SinanGabel/eac83a2f9d0ac64e2c9d4bd936be9313/3d302ee7b2667b70c8372e4f6ce4891811f2fb94#file-couchdb-2-0-install-L116):  
```
IOSchedulingClass=1
IOSchedulingPriority=0
```

## Sources:
 - https://www.beegfs.com/wiki/StorageServerTuning
 - https://tweaked.io/guide/kernel/
 - https://developer.couchbase.com/documentation/server/current/install/install-swap-space.html
 - http://www.tutorialspoint.com/unix_commands/ionice.htm
 - https://www.freedesktop.org/software/systemd/man/systemd.exec.html
