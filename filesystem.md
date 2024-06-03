# File System

## Ext4
[Ext4](https://en.wikipedia.org/wiki/Ext4) is the default file system currently used in our service.

## NILFS2
We had been operating with [NILFS2](https://nilfs.sourceforge.io/en/about_nilfs.html) applied to the cache (including all directories under `/var/cache`, such as the nginx cache), but discontinued its use as of June 3, 2024, due to the garbage collection (`nilfs_cleanerd`) not functioning correctly.

## Contact us
* ActivityPub [@catswords_oss@catswords.social](https://catswords.social/@catswords_oss)
* abuse@catswords.net
