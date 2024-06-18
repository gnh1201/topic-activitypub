# topic-activitypub
ActivityPub knowledge base from my open source projects

## My projects
* Caterpillar Proxy - The simple and parasitic web proxy with SPAM filter (formerly, php-httpproxy)
  * https://github.com/gnh1201/caterpillar
  * https://github.com/gnh1201/caterpillar/wiki/Fediverse
  * https://qiita.com/gnh1201/items/09f4081f84610db3a9d3 - K-Anonymity for Spam Filtering: Case with Mastodon, and Misskey
* ActivityPub (Fediverse) implementation for GNUBOARD5
  * https://github.com/gnh1201/gnuboard5-activitypub
* How to accelerate video trancoding (FFmpeg) in Mastodon
  * https://gist.github.com/gnh1201/1ba49e0e80a11237038900bf8abfa434
* Simple web API for extracting images/audio from video and converting audio/video/image files with ffmpeg.
  * https://github.com/gnh1201/ffmpeg-api
  * https://github.com/gnh1201/ffmpeg-api/blob/master/technical-notes.md
* Lingva Translate Gateway for Mastodon (LibreTranslate API compatible)
  * https://github.com/gnh1201/mastodon-lingva
* Social on the file - File reputation checker with Social media timeline
  * https://github.com/gnh1201/SocialOnTheFile
* ActivityPub over LoRaWAN (Draft)
  * Use [SenseCAP M2 Multi-Platform LoRaWAN Indoor Gateway(SX1302)](https://www.seeedstudio.com/SenseCAP-Multi-Platform-LoRaWAN-Indoor-Gateway-SX1302-EU868-p-5471.html)
  * Use [Dragino LA66 USB LoRaWAN Adapter](https://www.dragino.com/products/lora/item/232-la66-usb-lorawan-adapter.html)
  * Also, Please check `#caterpillar_proxy` hashtag in catswords.social

## Thirdparties
* [nginx](https://nginx.org/)
* [Cloudflare WARP (Zero Trust)](https://one.one.one.one/)
  * You can benefit from improved data transfer (federation) speeds between different nodes.
* [Cloudflare Tunnel (Argo Smart Routing)](https://www.cloudflare.com/products/tunnel/)
  * You can benefit from improved data transfer (federation) speeds between different nodes as well as website response speeds.
* [Cloudflare R2](https://www.cloudflare.com/ko-kr/developer-platform/r2/)
  * As an object storage service. In my case, I use it to save and share a server settings (certificates, configuration files, etc.) between my other servers.
* [acme.sh](https://github.com/acmesh-official/acme.sh)
  * SSL certificate issuance
* [squid-cache](https://www.squid-cache.org/)
  * If you're considering creating a proxy server yourself, you might want to consider squid-cache.
* Private VPN software: [OpenVPN](https://openvpn.net/), [WireGuard](https://www.wireguard.com/), [Shadowsocks](https://shadowsocks.org/), etc.
  * By appropriately utilizing private VPN software, you can configure network structures such as Cloud DMZ that can help in cost-saving.
* [SendGrid](https://sendgrid.com/)
  * Integrate fast and explore features with 100 emails/day forever.
* [Honeypots](honeypots.md)
* [Image stocks](https://policy.catswords.social/stock_images.html)

## Contact me
* ActivityPub [@catswords_oss@catswords.social](https://catswords.social/@catswords_oss)
* abuse@catswords.net
