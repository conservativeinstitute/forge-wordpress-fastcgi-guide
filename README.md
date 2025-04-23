# FastCGI Cache Setup for WordPress on Laravel Forge

Enable high-performance FastCGI page caching for WordPress on Laravel Forge NGINX servers. This setup bypasses PHP and MySQL for anonymous users, improving speed and reducing server load.

## 1. Create Cache Directory

```bash
sudo mkdir -p /var/cache/nginx/wordpress
sudo chown -R www-data:www-data /var/cache/nginx/wordpress
```

## 2. Update `/etc/nginx/nginx.conf`

Inside the `http {}` block:

### FastCGI Cache Definition

```nginx
fastcgi_cache_path /var/cache/nginx/wordpress levels=1:2 keys_zone=WORDPRESS:100m inactive=60m use_temp_path=off;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout invalid_header http_500;
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
```

### Performance and Security Settings

```nginx
events {
    worker_connections 4096;
}

http {

    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    types_hash_max_size 2048;

    server_names_hash_bucket_size 128;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Timeout Settings
    keepalive_timeout 30;
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;

    # Buffer & Body Limits
    client_body_buffer_size 32K;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # File Descriptor & Static File Cache
    open_file_cache max=10000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # SSL Settings
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # Logging
    access_log /var/log/nginx/access.log;

    # Gzip
    gzip on;

    # Virtual Hosts
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

## 3. Update Site-Specific NGINX Config

From the Forge site dashboard, Edit Files > Nginx Config.

### Cache Bypass Logic

Add this inside the `server {}` block, before any `location` blocks:

```nginx
set $skip_cache 0;
if ($request_method = POST) {
    set $skip_cache 1;
}
if ($query_string != "") {
    set $skip_cache 1;
}
if ($request_uri ~* "/wp-admin/|/xmlrpc.php|wp-.*.php|/feed/|index.php|sitemap(_index)?.xml") {
    set $skip_cache 1;
}
if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_no_cache|wordpress_logged_in") {
    set $skip_cache 1;
}
```

### PHP Location Block

Update or replace your existing `location ~ \.php$` block:

```nginx
location ~ \.php$ {
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

    fastcgi_cache_bypass $skip_cache;
    fastcgi_no_cache $skip_cache;
    fastcgi_cache WORDPRESS;
    fastcgi_cache_valid 200 301 302 60m;

    add_header X-Fastcgi-Cache $upstream_cache_status;
}
```

## 4. PHP-FPM Pool Tuning (Optional)

Edit `/etc/php/8.3/fpm/pool.d/www.conf`:

```ini
request_terminate_timeout = 30s

pm.max_children = 20
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6
```

## 5. Permissions and Group Access

Ensure correct permissions:

```bash
sudo chown -R www-data:www-data /var/cache/nginx/wordpress
sudo find /var/cache/nginx/wordpress -type d -exec chmod 755 {} \;
sudo find /var/cache/nginx/wordpress -type f -exec chmod 644 {} \;
sudo usermod -a -G www-data forge
sudo systemctl restart php8.3-fpm
```

## 6. Reload NGINX and Confirm Cache

Test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Verify cache status:

```bash
curl -I -H "Cookie:" https://yourdomain.com
```

Expected output:

```
x-fastcgi-cache: MISS
```

Then run it again:

```
x-fastcgi-cache: HIT
```

## 7. WordPress Integration with Nginx Helper Plugin

Install the [Nginx Helper Plugin](https://wordpress.org/plugins/nginx-helper/) and add the following to `wp-config.php`:

```php
define('RT_WP_NGINX_HELPER_CACHE_PATH', '/var/cache/nginx/wordpress');
define('RT_WP_NGINX_HELPER_CACHE_METHOD', 'fastcgi');
define('RT_WP_NGINX_HELPER_CACHE_PURGE_METHOD', 'unlink_files');
define('NGINX_HELPER_LOG', true);
define('NGINX_HELPER_LOG_FILES', 1);
```

This allows auto-purging on post updates, comment actions, and more.

Lastly, configure the plugin settings:

### Purging Options

- Enable Purge ✅
- Preload Cache ❌

### Caching Method

- nginx Fastcgi cache ✅
- Redis ❌

### Purge Method

- Using a GET request to PURGE/url ❌
- Delete local server cache files ✅

### Purging Conditions

- Enable all

### Debug Options

- Enable logging ✅
- Enable Nginx Timestamp in HTML ✅

### Logging Options

- Leave defaults

## Complete

Your WordPress site is now running with NGINX FastCGI full-page caching and optimized performance on Laravel Forge infrastructure.

You can add purging on post update or cache analysis endpoints as optional enhancements.
