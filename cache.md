# Explain about caching in an ActivityPub server

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

## Network Acceleration

### Cloudflare CDN (Inbound)
Cloudflare supports caching sizes up to 100MB per file (on the free plan), so integrating it with existing cloud infrastructure can lead to significant network acceleration effects.

### Cloudflare WARP with Squid Cache (Outbound)
You can accelerate your network by enabling the proxy mode of [Cloudflare WARP](https://one.one.one.one/) (please refer to the official documentation on how to enable it) and integrating it with [Squid Cache](https://www.squid-cache.org/). Outbound network traffic occurs when communicating with ActivityPub relays, performing message validation, etc. An example of a suitable `squid.conf` configuration is as follows:

```
cache_peer 127.0.0.1 parent 40000 0 no-query default
cache_peer_access 127.0.0.1 allow !localnet
never_direct deny localnet
never_direct allow all
prefer_direct on
http_access allow localnet
```

I observed that as outbound requests become more frequent, the resource demands of Cloudflare WARP's process, `warp-svc`, increase significantly. If performance improvement is necessary, you can use Squid Cache to offload proxy operations to a different machine.

### Splited edge
Regions with peering conflicts with CDN providers offering free plans may require a splitted edge. This approach requires setting up DNS subdomains and installing and configuring [geoipupdate (MaxMind)](https://github.com/maxmind/geoipupdate) along with the [ngx_http_geoip2_module](https://github.com/leev/ngx_http_geoip2_module) (an nginx extension module).

Below is an example of an nginx configuration that can be applied in such situations. This example assumes that paths using the CDN have the "www" subdomain (e.g., www.example.org), while paths not using the CDN use the default domain (e.g., example.org).

* nginx.conf

```
map $remote_addr $forwarded_remote_addr {
    default $remote_addr;
    127.0.0.1 $http_x_forwarded_for;
}

map $http_referer $http_referer_host {
    ~^.*://([^?/]+).*$ $1;
}

geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 60m;
    $geoip2_metadata_country_build metadata build_epoch;
    $geoip2_data_country_code default=UND source=$forwarded_remote_addr country iso_code;
    $geoip2_data_country_name country names en;
}

map $geoip2_data_country_code $allowed_country {
    default no;
    KR yes;
}
```

* [YOUR_WEBSITE].conf

```
server {
    server_name example.org www.example.org;

    # (...omitted...)

    # Set alternative domains.
    set $primary_proxy_host "example.org";
    set $secondary_proxy_host "www.example.org";

    # Check is it connect from an outside country.
    set $route_cdn yes;
    if ($host = $primary_proxy_host) {
        set $route_cdn $allowed_country;
    }
    # Websocket does not allow redirect requests, so it is directly linked.
    if ($http_upgrade = "websocket") {
        set $route_cdn yes;
    }
    # Redirect if all conditions are satisfied.
    if ($route_cdn = no) {
        return 307 https://$secondary_proxy_host$request_uri;
    }

    # (...omitted...)

    # when use an alternative domains
    sub_filter_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/x-icon application/activity+json application/ld+json;
    sub_filter_once off;
    sub_filter '/$primary_proxy_host' '/$host';
    sub_filter '\\/$primary_proxy_host' '\\/$host';

    # (...omitted...)
}
```

## Filesystem

### Ext4
[Ext4](https://en.wikipedia.org/wiki/Ext4) is the default file system currently used in my service.

### NILFS2
I had been using [NILFS2](https://nilfs.sourceforge.io/en/about_nilfs.html) for the cache (including all directories under `/var/cache`, such as the Nginx cache), but discontinued its use as of June 3, 2024, due to the garbage collection (`nilfs_cleanerd`) not functioning correctly.

## Any questions?
- abuse@catswords.net
- ActivityPub [@catswords_oss@catswords.social](https://catswords.social/@catswords_oss)
- [Join Catswords on Microsoft Teams](https://teams.live.com/l/community/FEACHncAhq8ldnojAI)
