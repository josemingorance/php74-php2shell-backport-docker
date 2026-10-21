# php:7.4-fpm with the official PHP2Shell backport

A reproducible Docker build of **PHP 7.4.33 + the official security backport for
CVE-2026-109618 / CVE-2026-109619 ("PHP2Shell")**, the unauthenticated remote code
execution flaws in PHP core fixed on 2026-10-21.

PHP 7.4 is end of life and will not get a 7.4.34. Because of the severity, the PHP
security team published minimal source backports for 7.4, 8.0 and 8.1 as git branches
(`PHP-7.4-CVE-2026-109618-109619`, commit `712e60f9e`, "Avoid state corruption via
bailout", 17 files), explicitly "to help people building from source". There is no
tarball and no official image. This repository is what "building from source" turned
out to require in October 2026.

If you can upgrade to 8.2+, do that instead. This is for the installations that cannot
upgrade today.

## What is in here

- `Dockerfile`: the last official docker-library Dockerfile for `php:7.4-fpm`
  (`7.4/bullseye/fpm` at docker-library/php@1361ba95) with four changes, each marked
  in the file.
- `docker-php-source`: the upstream helper plus one line that applies the backport
  diff on every extraction, so `docker-php-ext-install` also builds against patched
  sources.
- `docker-php-ext-*`, `docker-php-entrypoint`: upstream helpers, unchanged.
- `php2shell-CVE-2026-109618-109619-7.4.diff`: the official backport, verbatim from
  `https://github.com/php/php-src/compare/PHP-7.4...PHP-7.4-CVE-2026-109618-109619.diff`.
  It applies cleanly to the 7.4.33 release tarball (17 files, no rejects), which is the
  tip of the PHP-7.4 branch.

## The three things that break a 7.4 build today

These are not related to the vulnerability. They are why `docker build` of the
unmodified 2022 Dockerfile fails in 2026, and why a "just apply the patch" plan takes a
day instead of an hour.

1. **Debian bullseye is archived.** `deb.debian.org` still serves the package index
   but returns 404 for the `bullseye-security` packages. The Dockerfile rewrites
   `/etc/apt/sources.list` to `archive.debian.org` before the first `apt-get` and
   drops `bullseye-updates`.
2. **PHP 7.4's bundled PEAR can no longer read pecl.php.net's REST index.**
   `pecl install redis` fails with `No releases available for package
   "pecl.php.net/redis"` or `Package "redis" Version "6.3.0" does not have REST xml
   available`, while the same command works from a PHP 8 image. Download the exact
   tarball and install it from the file instead:

   ```dockerfile
   RUN curl -fsSL -o /tmp/redis.tgz https://pecl.php.net/get/redis-6.0.2.tgz \
    && pecl install /tmp/redis.tgz && rm /tmp/redis.tgz \
    && docker-php-ext-enable redis
   ```

3. **Latest extension releases no longer target 7.4.** Pin the versions you were
   already running (`pecl list` inside your current container tells you). Known to
   build on this image: `redis-6.0.2`, `redis-6.3.0`, `memcached-3.2.0`,
   `memcached-3.4.0`, `grpc-1.64.1`.

## Verifying the backport is in

`php -v` prints `7.4.33` with or without the patch, so it proves nothing. The backport
commit adds two exported symbols; the build fails unless one of them is present:

```sh
docker run --rm php74-backport nm -D /usr/local/bin/php | grep ' T zend_mm_safe_error_ex'
docker run --rm php74-backport cat /usr/local/etc/php/php2shell-backport
```

## Build and use

```sh
docker build -t php74-backport .
```

Then use it exactly like `php:7.4-fpm`:

```dockerfile
FROM php74-backport
RUN docker-php-ext-install pdo_mysql opcache
```

Build time is dominated by the PHP compile (about 2 minutes on an Apple M-series,
longer on CI runners). The image is the same size as the official one.

## What the backport does (short version)

- Memory-limit failures no longer leave half-initialised request state behind
  (`zend_mm_safe_error_ex`), and request globals that could survive a failed request
  are reset.
- The session upload-progress handler, the demonstrated trigger for CVE-2026-109619,
  is disabled unconditionally in `ext/session/session.c`.
- The multipart parser (`main/rfc1867.c`) keeps only the two headers it uses per
  part, ignores duplicates, and gains the `max_multipart_body_parts` limit.

Read the advisories for the full picture:
[GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w)
and [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp).
The official announcement: https://www.php.net/archive/2026.php#2026-10-21-5

## Licenses

The Dockerfile and helper scripts come from
[docker-library/php](https://github.com/docker-library/php) (MIT). The diff is PHP
source code under the [PHP License](https://www.php.net/license/). The changes made
here are released under the MIT License (see `LICENSE`).
