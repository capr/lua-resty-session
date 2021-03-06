# lua-resty-session

**lua-resty-session** is a session library for OpenResty implementing [Secure Cookie Protocol](http://www.cse.msu.edu/~alexliu/publications/Cookie/cookie.pdf).

## Hello World with lua-resty-session

```nginx
http {
    server {
        listen       8080;
        server_name  localhost;
        default_type text/html;
        location / {
            content_by_lua '
                ngx.say("<html><body><a href=/start>Start the test</a>!</body></html>")
            ';
        }
        location /start {
            content_by_lua '
                local session = require("resty.session").start()
                session.data.name = "OpenResty Fan"
                session:save()
                ngx.say("<html><body>Session started. ",
                        "<a href=/test>Check if it is working</a>!</body></html>")
            ';
        }
        location /test {
            content_by_lua '
                local session = require("resty.session").start()
                ngx.say("<html><body>Session was started by <strong>",
                        session.data.name or "Anonymous",
                        "</strong>! <a href=/destroy>Destroy the session</a>.</body></html>")
            ';
        }
        location /destroy {
            content_by_lua '
                local session = require("resty.session").start()
                session:destroy()
                ngx.say("<html><body>Session was destroyed. ",
                        "<a href=/check>Is it really so</a>?</body></html>")
            ';
        }
        location /check {
            content_by_lua '
                local session = require("resty.session").start()
                ngx.say("<html><body>Session was really destroyed, you are known as ",
                        "<strong>",
                        session.data.name or "Anonymous",
                        "</strong>! <a href=/>Start again</a>.</body></html>")
            ';
        }
    }
}
```

## Installation

Just place [`session.lua`](https://github.com/bungle/lua-resty-session/blob/master/lib/resty/session.lua) somewhere in your `package.path`, preferably under `resty` directory. If you are using OpenResty, the default location would be `/usr/local/openresty/lualib/resty`.

### Using LuaRocks or MoonRocks

If you are using LuaRocks >= 2.2:

```Shell
$ luarocks install lua-resty-session
```

If you are using LuaRocks < 2.2:

```Shell
$ luarocks install --server=http://rocks.moonscript.org moonrocks
$ moonrocks install lua-resty-session
```

MoonRocks repository for `lua-resty-session` is located here: https://rocks.moonscript.org/modules/bungle/lua-resty-session.

## About The Defaults

`lua-resty-session` does by default set session only cookies (non-persistent, `HttpOnly`) so that
the cookies are not readable from Javascript (not subjectible to XSS in that matter). It will also set
`Secure` flag by default when the request was made via SSL/TLS connection. Cookies send via SSL/TLS
don't work when sent via HTTP and vice-versa. By default the HMAC key is generated from session id (random
bytes generated with OpenSSL), expiration time, unencrypted data, and Nginx variables `ssl_session_id`
(if requested with TLS/SSL), `http_user_agent` and `scheme`. You may also configure it to use
`remote_addr` as well by setting `set $session_check_addr on;` (but this may be problematic
with clients behind proxies or NATs that change the remote address between requests).

The data part is encrypted with AES-algorithm (by default it uses OpenSSL `EVP_aes_256_cbc` and
`EVP_sha512` functions that are provided with `lua-resty-string`. They come preinstalled with
the default OpenResty bundle. The `lua-resty-session` library is not tested with all the
`resty.aes` functions (but the defaults are tested to be working). Please let me know or contact
`lua-resty-string` project if you hit any problems with different algorithms.

Session identifier length is by default 16 bytes (randomly generated data with OpenSSL
`RAND_pseudo_bytes` function). The server secret is also generated by default with this same
function, and it's length is determined by calculating the used `$session_cipher_size` divided
by 8 (so by default it uses 32 bytes). This will work until Nginx is restarted, but you might want
to consider setting your own secret using `set $session_secret 623q4hR325t36VsCD3g567922IC0073T;`,
for example (this will work in farms installations as well, but you are then responsible for
rotating the secret). On farm installations you should also configure other session configuration
variables the same on all the servers in the farm.

Cookie parts are encoded with cookie safe Base64 encoding. Before encrypting and encoding the data
part, the data is serialized with JSON encoding (so you can use basic Lua types in data, and expect
to receive them back as the same Lua types). JSON encoding is done by the bundled OpenResty cJSON
library (Lua cJSON). Cookie's path scope is by default `/` (meaning that it will be send to all paths
in the server, and the Domain scope is the current host determined by `host` Nginx variable (meaning
that the browser will only send cookie back to the same host from which it got it on a first place).

There is no state held in server (except the configuration directives), and all the data is stored
in the cookie as specified in Secure Cookie Protocol paper. In the future, the server side backend
could be added to this module as well (all contributions are welcomed).

## Lua API

### Functions and Methods

#### table session.start(opts or nil)

With this function you can start a new session. It will create a new session Lua `table` on each call.
Right now you should only start session once per request as calling this function repeatedly will overwrite the previously
started session cookie and session data. This function will return a new session `table` as a result. If the session cookie
is supplied with user's HTTP(S) client then this function validates the supplied session cookie. If validation
is successful, the user supplied session data will be used (if not, a new session is generated with empty data).
You may supply optional session configuration variables with `opts` argument, but be aware that many of these
will only have effect if the session is a fresh session (i.e. not loaded from user supplied cookie). This function
does also manage session cookie renewing configured with `$session_cookie_renew`. E.g. it will send a new cookie
with a new expiration time if the following is met `session.expires - now < session.cookie.renew`.

```lua
local session = require("resty.session").start()
-- Set some options (overwriting the defaults or nginx configuration variables)
local session = require("resty.session").start{ identifier = { length = 32 }}
```

#### boolean session:regenerate(flush or nil)

This function regenerates a session. It will generate a new session identifier and optionally flush the
session data if `flush` argument evaluates `true`. It will automatically `session:save` which
means that a new expires flag is set on the cookie, and the data is encrypted with the new parameters. With
client side sessions (server side sessions are not yet supported) this overwrites the current cookie with
a new one (but it doesn't invalidate the old one as there is no state held on server side - invalidation
actually happens when the cookie's expiration time is not valid anymore). This function returns a boolean
value if everything went as planned (you may assume that it is always the case).

```lua
local session = require("resty.session").start()
session:regenerate()
-- Flush the current data
session:regenerate(true)
```

#### boolean session:save()

This function saves the session and sends a new cookie to client (with a new expiration time and encrypted data).
You need to call this function whenever you want to save the changes made to `session.data` table. It is
advised that you call this function only once per request (no need to encrypt and set cookie many times).
This function returns a boolean value if everything went as planned (you may assume that it is always the case).

```lua
local session = require("resty.session").start()
session.data.uid = 1
session:save()
```

#### boolean session:destroy()

This function will immediately set session data to empty table `{}`. It will also send a new cookie to
client with empty data and Expires flag `Expires=Thu, 01 Jan 1970 00:00:01 GMT` (meaning that the client
should remove the cookie, and not send it back again). This function returns a boolean value if everything went
as planned (you may assume that it is always the case).

```lua
local session = require("resty.session").start()
session:destroy()
```

### Fields

#### string session.id

`session.id` holds the current session id. By default it is 16 bytes long (raw binary bytes).
It is automatically generated.

#### number session.identifier.length

`session.identifier.length` holds the length of the `session.id`. By default it is 16 bytes.
This can be configured with Nginx `set $session_identifier_length 16;`.

#### string session.key

`session.key` holds the HMAC key. It is automatically generated. Nginx configuration like
`set $session_check_ua on;`, `set $session_check_scheme on;` and `set $session_check_addr on;`
 will have effect on the generated key.

#### table session.data

`session.data` holds the data part of the session cookie. This is a Lua `table`. `session.data`
is the place where you store or retrieve session variables. When you want to save the data table,
you need to call `session:save` method.

**Setting session variable:**

```lua
local session = require("resty.session").start()
session.data.uid = 1
session:save()
```

**Retrieving session variable (in other request):**

```lua
local session = require("resty.session").start()
local uid = session.data.uid
```

#### number session.expires

`session.expires` holds the expiration time of the session (expiration time will be generated when
`session:save` method is called).

#### string session.secret

`session.secret` holds the secret that is used in keyed HMAC generation.

#### number session.cookie.renew

`session.cookie.renew` holds the minimun seconds until the cookie expires, and renews cookie automatically
(i.e. sends a new cookie with a new expiration time according to `session.cookie.lifetime`). This can be configured
with Nginx `set $session_cookie_renew 600;` (600 seconds is the default value).

#### number session.cookie.lifetime

`session.cookie.lifetime` holds the cookie lifetime in seconds in the future. By default this is set
to 3,600 seconds. This can be configured with Nginx `set $session_cookie_lifetime 3600;`. This does not
set cookie's expiration time as this library will only use session only (and `HttpOnly` cookies). That
means that cookies are not persistent and they are deleted when the client browser is closed.

#### string session.cookie.path

`session.cookie.path` holds the value of the cookie path scope. This is by default permissive `/`. You
may want to have a more specific scope if your application resides in different path (e.g. `/forums/`).
This can be configured with Nginx `set $session_cookie_path /forums/;`.

#### string session.cookie.domain

`session.cookie.domain` holds the value of the cookie domain. By default this is automatically set using
Nginx variable `host`. This can be configured with Nginx `set $session_cookie_domain openresty.org;`.
For `localhost` this is omitted.

#### boolean session.cookie.secure

`session.cookie.secure` holds the value of the cookie `Secure` flag. meaning that when set the client will
only send the cookie with encrypted TLS/SSL connection. By default the `Secure` flag is set on all the
cookies where the request was made through TLS/SSL connection. This can be configured and forced with
Nginx `set $session_cookie_secure on;`.

#### boolean session.cookie.httponly

`session.cookie.httponly` holds the value of the cookie `HttpOnly` flag. By default this is enabled,
and I cannot think of an situation where one would want to turn this off. By keeping this on you can
prevent your session cookies access from Javascript and give some safety of XSS attacks. If you really
want to turn this off, this can be configured with Nginx `set $session_cookie_httponly off;`.

#### number session.cipher.size

`session.cipher.size` holds the size of the cipher (`lua-resty-string` supports AES in `128`, `192`, and `256` bits key sizes). See `aes.cipher` function in `lua-resty-string` for more information. By default this will use `256` bits key size. This can be configured with Nginx `set $session_cipher_size 256;`.

#### string session.cipher.mode

`session.cipher.mode` holds the mode of the cipher. `lua-resty-string` supports AES in `ecb`, `cbc`, `cfb1`, `cfb8`, `cfb128`, `ofb`, and `ctr` modes (ctr mode is not available with 256 bit keys). See `aes.cipher` function in `lua-resty-string` for more information. By default `cbc` mode is used. This can be configured with Nginx `set $session_cipher_mode cbc;`.

#### function session.cipher.hash

`session.cipher.hash` is used in ecryption key, and iv derivation (see: OpenSSL [EVP_BytesToKey](https://www.openssl.org/docs/crypto/EVP_BytesToKey.html)). By default `sha512` is used but `md5`,
`sha1`, `sha224`, `sha256`, and `sha384` are supported as well in `lua-resty-string`. This can be configured with Nginx `set $session_cipher_hash sha512;`.

#### number session.cipher.rounds

`session.cipher.rounds` can be used to slow-down the encryption key, and iv derivation. By default this is set to `1` (the fastest). This can be configured with Nginx `set $session_cipher_rounds 1;`.

#### boolean session.check.ua

`session.check.ua` is additional check to validate that the request was made with the same user-agen browser string
as where the original cookie was delivered. This check is enabled by default.

#### boolean session.check.addr

`session.check.addr` is additional check to validate that the request was made from the same remote ip-address
as where the original cookie was delivered. This check is disabled by default.

#### boolean session.check.addr

`session.check.scheme` is additional check to validate that the request was made using the same protocol 
as the one used when the original cookie was delivered. This check is enabled by default.

#### Additional checks that are not configurable

`lua-resty-session` will always check on TLS/SSL connection whether the cookie was send with the same `ssl_session_id`
that was used when the cookie was originally delivered.

## Nginx Configuration Variables

Here is a list of Nginx configuration variables that you can use to control `lua-resty-session`:

```nginx
set $session_name              session;
set $session_secret            623q4hR325t36VsCD3g567922IC0073T;
set $session_cookie_renew      600;
set $session_cookie_lifetime   3600;
set $session_cookie_path       /;
set $session_cookie_domain     openresty.org;
set $session_cookie_secure     on;
set $session_cookie_httponly   on;
set $session_cipher_mode       cbc;
set $session_cipher_size       256;
set $session_cipher_hash       sha512;
set $session_cipher_rounds     1;
set $session_check_ua          on;
set $session_check_scheme      on;
set $session_check_addr        off;
set $session_identifier_length 16;
```

## License

`lua-resty-session` uses two clause BSD license.

```
Copyright (c) 2014, Aapo Talvensaari
All rights reserved.

Redistribution and use in source and binary forms, with or without modification,
are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this
  list of conditions and the following disclaimer in the documentation and/or
  other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
```
