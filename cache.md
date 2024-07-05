# All about a cache in ActivityPub network

## Cache size
When operating an ActivityPub network, it is advisable to configure media caching in the cloud (AWS Lightsail, Vultr, etc.).

The media files shared on the ActivityPub network are mostly around 2MB in size. This is the size of the converted files, not the originals. A cache size of 2MB per file is appropriate.

Below are some examples of links where you can check the size of media files.

* Avatar: 92.3KB
  * https://catswords.social/system/accounts/avatars/109/419/749/463/281/498/original/33300cd69a1065be.png
* Emoji: 5.21KB
  * https://catswords.social/system/cache/custom_emojis/images/000/214/949/original/7cd4227785d8a56c.png
* Animated GIF, MP4 converted: 635KB
  * https://catswords.social/@gnh1201/112635700763233336
* Long (4 minutes) video, MP4 converted: 19MB
  * https://catswords.social/@gnh1201/112631390630236775
* Short video, MP4 converted: 1.3MB
  * https://catswords.social/@gnh1201/112631239913009057

This example can be utilized in devising network acceleration strategies.

## CDN(Content Delivery Network)

### Cloudflare
Cloudflare supports caching sizes up to 100MB per file (on the free plan), so integrating it with existing cloud infrastructure can lead to significant network acceleration effects.

## Filesystem

### Ext4
[Ext4](https://en.wikipedia.org/wiki/Ext4) is the default file system currently used in our service.

### NILFS2
We had been operating with [NILFS2](https://nilfs.sourceforge.io/en/about_nilfs.html) applied to the cache (including all directories under `/var/cache`, such as the nginx cache), but discontinued its use as of June 3, 2024, due to the garbage collection (`nilfs_cleanerd`) not functioning correctly.

## Any question?
- abuse@catswords.net
- ActivityPub [@catswords_oss@catswords.social](https://catswords.social/@catswords_oss)
