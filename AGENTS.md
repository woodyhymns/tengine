# AGENTS.md

## Cursor Cloud specific instructions

### Overview

This is **Tengine** — an enhanced fork of Nginx (based on nginx-1.6.2), developed by Alibaba/Taobao. It is a C-based HTTP/reverse-proxy web server built with `./configure && make && make install`.

### Build instructions

The standard build commands are in `README.markdown`. Key gotchas for modern Ubuntu (24.04+):

- **OpenSSL 1.0.2 required**: Tengine's SSL code uses internal OpenSSL structs that were made opaque in OpenSSL 1.1+. Download OpenSSL 1.0.2u source to `/tmp/openssl-1.0.2u` and pass `--with-openssl=/tmp/openssl-1.0.2u` to `./configure`.
- **Compiler warning flags**: Modern Clang/GCC treat some old C patterns as errors. Pass `--with-cc-opt="-Wno-error -Wno-int-conversion -Wno-incompatible-pointer-types -Wno-implicit-function-declaration -Wno-deprecated-declarations"` to `./configure`.
- **libxslt detection**: The feature detection test has an int-to-pointer assignment that newer compilers reject; the `-Wno-error` flag above resolves this.
- **`struct crypt_data`**: Modern libxcrypt removed the `current_salt` field. The build fix in `src/os/unix/ngx_user.c` guards the old glibc workaround with `!defined(CRYPT_DATA_INTERNAL_SIZE)`.

Full configure command:
```
./configure --enable-mods-static=all --with-ipv6 --with-http_spdy_module \
  --with-openssl=/tmp/openssl-1.0.2u \
  --with-cc-opt="-Wno-error -Wno-int-conversion -Wno-incompatible-pointer-types -Wno-implicit-function-declaration -Wno-deprecated-declarations"
make -j$(nproc)
sudo make install
```

Binary installs to `/usr/local/nginx/sbin/nginx`.

### Running Tengine

```
sudo /usr/local/nginx/sbin/nginx          # start
sudo /usr/local/nginx/sbin/nginx -s stop   # stop
sudo /usr/local/nginx/sbin/nginx -s reload # reload config
curl http://localhost/                      # verify
```

Tengine listens on port 80 by default. Config is at `/usr/local/nginx/conf/nginx.conf`.

### Running tests

The primary test suite is **nginx-tests** (Perl-based). The `test-nginx` (OpenResty-style) suite requires additional Perl modules and may segfault due to Lua module compatibility.

```
TEST_NGINX_BINARY=/usr/local/nginx/sbin/nginx \
  PERL5LIB=/workspace/tests/nginx-tests/nginx-tests/lib \
  prove /workspace/tests/nginx-tests/cases
```

Some tests (e.g., `resolver.t`) fail in sandboxed environments without DNS.

### Lint / style checking

- `contrib/stylechecker.py` is Python 2 only (not runnable on Python 3).
- Compiler warnings during `make` serve as the primary code quality check.

### System dependencies

```
sudo apt-get install -y build-essential libpcre3-dev libssl-dev zlib1g-dev \
  liblua5.1-0-dev lua5.1 libgd-dev libgeoip-dev libxslt1-dev libperl-dev perl
```
