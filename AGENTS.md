# AGENTS.md

## Cursor Cloud specific instructions

### Overview

This is **Tengine** (v2.2.0), an Nginx fork by Alibaba based on nginx/1.8.0. It is a C web server built from source using `./configure && make && make install`. The binary installs to `/usr/local/nginx/`.

### Build

The codebase targets **GCC** and **OpenSSL 1.0.2**. Key build caveats on modern Ubuntu 24.04:

- **Must use `CC=gcc`**: The default compiler `cc` may resolve to clang 18, which is too strict for this older C code.
- **OpenSSL 1.0.2 required**: The code uses OpenSSL 1.0.x APIs (stack-allocated `EVP_MD_CTX`, `CRYPTO_add`, `CRYPTO_LOCK_X509`, etc.) that were removed in OpenSSL 1.1+. Download source from `https://www.openssl.org/source/old/1.0.2/openssl-1.0.2u.tar.gz` and use `--with-openssl=/path/to/openssl-1.0.2u`.
- **GCC 13 warning suppression**: Pass `--with-cc-opt="-Wno-implicit-fallthrough -Wno-deprecated-declarations -Wno-cast-function-type -Wno-error -Dcurrent_salt=output"` to suppress warnings-as-errors from newer GCC.
- **`-Dcurrent_salt=output`**: Works around `struct crypt_data` in modern libxcrypt no longer having the `current_salt` field.

Full configure command:
```
CC=gcc ./configure --prefix=/usr/local/nginx \
  --enable-mods-static=all --with-ipv6 --with-http_spdy_module \
  --with-openssl=/tmp/openssl-1.0.2u \
  --with-cc-opt="-Wno-implicit-fallthrough -Wno-deprecated-declarations -Wno-cast-function-type -Wno-error -Dcurrent_salt=output"
```

### Running

```bash
sudo /usr/local/nginx/sbin/nginx          # start
sudo /usr/local/nginx/sbin/nginx -s stop   # stop
sudo /usr/local/nginx/sbin/nginx -s reload # reload config
/usr/local/nginx/sbin/nginx -V             # show version & modules
```

Config: `/usr/local/nginx/conf/nginx.conf`  
Document root: `/usr/local/nginx/html/`  
Listens on port 80 by default.

### Tests

Tests are Perl-based and live in `tests/`. Two test suites:

1. **nginx-tests** (upstream Nginx tests): Run from `tests/nginx-tests/nginx-tests/`:
   ```bash
   TEST_NGINX_BINARY=/usr/local/nginx/sbin/nginx prove -r *.t
   ```
   ~116/121 files pass; failures in `debug_connection.t`, `error_log.t`, `http_raw_uri_variable.t`, `ssl.t`, `syslog.t` are environment-specific.

2. **Tengine cases** (Tengine-specific tests): Run from `tests/nginx-tests/`:
   ```bash
   TEST_NGINX_BINARY=/usr/local/nginx/sbin/nginx prove cases/server_banner.t cases/string.t cases/limit_req_enhance.t cases/variable.t
   ```
   Note: `cases/` tests need a `lib/` symlink: `ln -sf ../nginx-tests/lib tests/nginx-tests/cases/lib`. Some tests (e.g., `concat.t`) expect DSO modules and will fail with a static-only build.

3. **Test::Nginx (OpenResty)**: Used by `tests/test-nginx/` tests. Install from `tests/test-nginx/test-nginx/` with `PERL_MM_USE_DEFAULT=1 perl Makefile.PL && make && sudo make install`.

### Lint

No dedicated linter. The build uses `-Werror` by default (overridden by `-Wno-error` in our cc-opt). To check for compile errors without `-Wno-error`, remove that flag from `--with-cc-opt`.

### System dependencies

Required apt packages: `gcc`, `make`, `libpcre3-dev`, `libssl-dev`, `libgd-dev`, `libgeoip-dev`, `libxslt1-dev`, `liblua5.1-0-dev`, `lua5.1`, `libperl-dev`, `zlib1g-dev`, `libxml2-dev`.

Perl test deps: `libtext-diff-perl`, `libtest-longstring-perl`, `libwww-perl`, `liburi-perl`, `liblist-moreutils-perl`.
