# File System

## NILFS2
We had been operating with [NILFS2](https://nilfs.sourceforge.io/en/about_nilfs.html) applied to the cache (including all directories under /var/cache, such as the nginx cache), but discontinued its use as of June 3, 2024, due to the garbage collection (nilfs_cleanerd) not functioning correctly.
