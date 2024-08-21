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
Cloudflare supports caching sizes of up to 100MB per file (on the free plan), so integrating it with existing cloud infrastructure can result in significant network acceleration. Additionally, using [cloudflared (aka. Zero Trust Access, Argo Tunnel)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) helps resolve some undocumented limitations, such as issues with routing priority varying by region.

### Cloudflare WARP with Squid Cache (Outbound)

You can accelerate your network by enabling [Cloudflare WARP](https://one.one.one.one/) ( and integrating it with [Squid Cache](https://www.squid-cache.org/). It supports a proxy mode that uses a specific port (default: 40000/TCP) and a WARP mode that functions like a VPN. Since the proxy mode is less efficient, it is recommended to use the WARP mode.

Outbound network traffic occurs when communicating with ActivityPub relays, performing message validation, etc. An example of a suitable `squid.conf` configuration is as follows:

#### In the operational server

```
cache_peer [x.x.x.x] parent 3128 0 no-query default
cache_peer_access [x.x.x.x] allow all
never_direct allow all
prefer_direct on
http_access allow localnet
```

#### In the proxy server (If Cloudflare WARP installed)

```
http_access allow localnet
```

I recommend separating the operation server and the proxy server. You can install Squid on each server to relay proxy requests, and set up redundancy so that the local connection is used automatically when Cloudflare WARP is unavailable.

### Splitted edge
[Regions with peering conflicts with CDN providers offering free plans](https://www.gameple.co.kr/news/articleView.html?idxno=208925) may require a splitted edge. This approach requires setting up DNS subdomains and installing and configuring [geoipupdate (MaxMind)](https://github.com/maxmind/geoipupdate) along with the [ngx_http_geoip2_module (an nginx extension module)](https://github.com/leev/ngx_http_geoip2_module).

Below is an example of an nginx configuration that can be applied in such situations. This example assumes that paths using the CDN have the "www" subdomain (e.g., www.example.org), while paths not using the CDN use the default domain (e.g., example.org).

* nginx.conf

```
map $remote_addr $forwarded_remote_addr {
    default $remote_addr;
    127.0.0.1 $http_x_forwarded_for;
}

#map $http_referer $http_referer_host {
#    ~^.*://([^?/]+).*$ $1;
#}

geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 60m;
    $geoip2_metadata_country_build metadata build_epoch;
    $geoip2_data_country_code default=UND source=$forwarded_remote_addr country iso_code;
    $geoip2_data_country_name country names en;
}

map $geoip2_data_country_code $allowed_country {
    default no;
    KR yes;    # South Korea
    #JP yes;    # Japan
}
```

* [YOUR_WEBSITE].conf

```
limit_conn_zone $binary_remote_addr zone=website_conn:20m;
#limit_conn_zone $proxy_add_x_forwarded_for zone=website_conn:20m;
limit_conn_status 429;

server {
    server_name example.org www.example.org;

    # (...omitted...)

    # Recover the real client IP
    real_ip_header X-Forwarded-For;
    set_real_ip_from 127.0.0.1;
    #real_ip_recursive on;

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
    # Allow if it is not a HTTP GET method
    if ($request_method != "GET") {
        set $route_cdn yes;
    }
    # Allow if it is not a image and not a browser
    set $is_image 0;
    if ($uri ~* "\.(jpg|jpeg|gif|png|bmp|webp|svg|tiff|tif|heic|avif|raw|psd|ai|eps|pdf)$") {
        set $is_image 1;
    }
    set $is_browser 0;
    if ($http_user_agent ~* "Mozilla|Chrome|Safari|Opera|Edge|Trident|MSIE") {
        set $is_browser 1;
    }
    set $is_image_is_browser "$is_image$is_browser";
    if ($is_image_is_browser = "00") {
        set $route_cdn yes;
    }
    # Not allow if it is an audio or a video
    set $is_video 0;
    if ($uri ~* "\.(mp4|mkv|avi|mov|wmv|flv|webm|mpeg|m4v|3gp|ogg|mp3|wav|flac|aac|m4a|wma|alac|aiff|opus)$") {
        set $is_video 1;
    }
    set $is_video_host "$is_video$host";
    if ($is_video_host = "1$primary_proxy_host") {
        set $route_cdn no;
    }
    # Allow robots.txt (Robots Exclusion Standard)
    if ($request_uri ~* "/robots\.txt$") {
        set $route_cdn yes;
    }
    # Allow RSS (Really Simple Syndication)
    if ($request_uri ~* "\.(rss)$") {
        set $route_cdn yes;
    }
    # Redirect if all conditions are satisfied.
    if ($route_cdn = no) {
        return 307 https://$secondary_proxy_host$request_uri;
    }

    # Define a variable to control caching
    set $cache_bypass 0;

    # Check if request path or referer matches certain patterns
    if ($request_uri ~* "^(/auth/|/oauth/|/explore|/getting-started)") {
        set $cache_bypass 1;
    }
    if ($http_referer ~* "^(/auth/|/oauth/)") {
        set $cache_bypass 1;
    }

    location / {
        proxy_pass             https://your-own-server;
        proxy_set_header       Host $host;
        proxy_set_header       X-Real-IP $remote_addr;
        proxy_set_header       X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header       X-Forwarded-Host $host;
        proxy_set_header       X-Forwarded-Proto $scheme;
        proxy_ignore_headers   X-Accel-Expires Expires Cache-Control;
        proxy_buffering        on;
        proxy_cache            STATIC;
        proxy_cache_valid      200 4s;
        proxy_cache_use_stale  error timeout invalid_header updating
                               http_500 http_502 http_503 http_504;
        if ($cache_bypass = 1) {
            add_header             X-Proxy-Cache $upstream_cache_status;
            add_header             Cache-Control "public, max-age=4, s-maxage=4";
        }
        if ($cache_bypass = 0) {
            add_header             Cache-Control "no-cache, no-store, must-revalidate";
        }
        proxy_ignore_headers   X-Accel-Expires Expires Cache-Control;

        # Prevent data exposure
        proxy_cache_key        "$http_cookie$http_x_auth_token$scheme$primary_proxy_host$request_uri";
        proxy_no_cache         $cache_bypass;
        proxy_cache_bypass     $cache_bypass;

        # Enable the file upload buffering (Default: on)
        #proxy_request_buffering on;

        # Set limit of concurrent connection
        limit_conn mastodon_limit 30;
    }

    # when use an alternative domains
    # Do not apply 'application/activity+json' or 'application/ld+json'. There have been reports that corrupted Activities cause issues with the moderation features of the Misskey application.
    sub_filter_types text/plain text/css text/xml application/xml application/xml+html;
    sub_filter_once off;
    sub_filter '/$primary_proxy_host' '/$host';
    sub_filter '\\/$primary_proxy_host' '\\/$host';

    # (...omitted...)
}
```

### Splitted Edge Configuration Using Multiple CDN Services
CDN services can be classified into two types: a dynamic method that uses an Interactive [Reverse Proxy](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/) and a static (Non-interactive Reverse Proxy, aka. PushCDN) method that does not. Cloudflare is a representative example of a dynamic method, while many other CDN services are likely to use the static method.

By combining these two types of services, you can implement what is known as a "splitted edge." This involves first resolving the user's region (determining which country the user belongs to) using the static method and then serving content using the dynamic method.

#### Bunny CDN ([bunny.net](https://bunny.net)) configurations
- Caching/General
    - Vary Cache (If not enabled, a response could be compromised.)
        - Checked `URL Query String`
        - Checked `Cookie Value`
- Edge Rules
    - **Downgrade CSP(Content-Security-Policy) for specific regions**
        - **Actions:**
            - Set Response Header: `Content-Security-Policy base-uri 'none'; default-src 'none'; frame-ancestors 'none'; font-src 'self' https://example.org https://*.example.org; img-src 'self' https: data: blob: https://example.org https://*.example.org; style-src 'self' https://example.org https://*.example.org 'unsafe-inline'; media-src 'self' https: data: https://example.org https://*.example.org; frame-src 'self' https: https://example.org https://*.example.org; manifest-src 'self' https://example.org https://*.example.org; form-action 'self' https://example.org https://*.example.org; child-src 'self' blob: https://example.org https://*.example.org; worker-src 'self' blob: https://example.org https://*.example.org; connect-src 'self' data: blob: https://example.org https://*.example.org wss://www.example.org; script-src 'self' https://example.org https://*.example.org 'wasm-unsafe-eval' 'unsafe-inline' 'unsafe-eval'`
        - **Conditions:**
            - **IF:**
                - ALL Condition matches:
                    - ANY Country Code: `KR`
                    - ANY Response Header: `Content-Security-Policy *`
    - **Bypass Cache if use Authentication**
        - **Actions:**
            - Bypass Perma-Cache
        - **Conditions:**
            - **IF:**
                - ANY Condition matches:
                    - ANY Request URL: `https://example.org/auth/*`, `https://example.org/oauth/*`, `https://example.org/`, `https://example.org/explore`, `https://example.org/getting-started`
                    - ANY Request Header: `Referer https://example.org/auth/*`, `Referer https://example.org/oauth/*`

#### NGINX configurations
Static proxies do not support WebSocket. Therefore, you may need to apply the following configuration.

```
sub_filter 'wss://$primary_proxy_host' 'wss://$secondary_proxy_host';    # (Be careful!) Bypass a WebSocket requests to dynamic proxy service
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
