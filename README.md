# music-builder
Musicoin Build Server

This solution right now used as build server for [Musicoin Desktop Wallet](https://github.com/Musicoin/desktop)

But actually could be used as example how to setup builder server for any NW.js application.

By default we presume that you'll able to install all used software by yourself, and only harder parts are covered.

### Details

- [Web2Exe](https://github.com/jyapayne/Web2Executable) used to generate NW.js project, you can locate it in `/tools/Web2ExeLinux-CMD`
- src folder holds [MDW repository](https://github.com/Musicoin/desktop) and [music-wallet-modules](https://github.com/cryptofuture/music-wallet-modules).


Reason not to use submodules is pretty easy, we just clone repositories we use locally and do updates with `git pull` - that allows not to make submodules updates in builder repository itself. 

Simple extra repository solution to hold Windows and MacOS `node_modules` prefered, since we don't do any auth on builder application itself, or share login passwords (which could always leak outside the team) to allow file uploads. Instead we just allow to publish update by the team members in the repository in the friendly for every developer manner.

- `tools/7z` in fact never used but containts stubs for 7-Zip to allow sfx archives generation, that could be launched under Windows. Stubs should be located in `/usr/lib/p7zip/` or 7z will never find a stub. You could find those stubs in 7-Zip for windows installation files. We use "GUI" version of stubs, instead on console version, that runs in the cmd.

- Symlink done from `/output/tmp/` to `/src/desktop/output` this allows to avoid bug, when Web2Exe couldn't find relative path with `--output-dir`

- `last-commit.log` file includes last commit build was done from. `last-modules-commit.log` file includes last commit for [music-wallet-modules](https://github.com/cryptofuture/music-wallet-modules) repository. And `build-time.log` contains build start and end time.

### Running:

We run with `su -s /bin/bash www-data -c '/var/www/music-builder/bin/builder'` this allows to run script for a user without actula shell. Force scripts used when user need to rebuild some build manually.

Or cronjob example : `1 */12 * * * su -s /bin/bash www-data -c '/var/www/music-builder/bin/builder'`

Nginx example:

```
server {
    listen 80;
    server_name builder.musicoin.org;

    root /var/www/music-builder/output/archive/final;
    error_log /var/log/nginx/music-builder.log;
    access_log /var/log/nginx/music-builder.log;
    include /etc/nginx/expires.conf;
    autoindex on;
}
```

After you can proxy it on balancer with tls for example:

```
proxy_cache_path /mnt/nginx/music levels=1:2 keys_zone=music:10m max_size=500m use_temp_path=off inactive=10m;

upstream builder.musicoin.org  {
	server  10.x.x.x:80;
}

server {
	listen 443 ssl http2;
	server_name builder.musicoin.org;
	ssl_certificate /etc/nginx/ssl/music-builder.crt;
	ssl_certificate_key /etc/nginx/ssl/music-builder.key;

	error_log /var/log/nginx/music-builder.log;
	access_log /var/log/nginx/music-builder.log;

	location / {
	proxy_cache music;
	add_header X-Proxy-Cache $upstream_cache_status;
	proxy_cache_revalidate on;
	proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
	proxy_cache_lock on;

	proxy_pass http://builder.musicoin.org/;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	charset utf-8;
	}
}
```





