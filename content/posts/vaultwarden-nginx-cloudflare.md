+++
title = 'Self-hosting Vaultwarden behind nginx and Cloudflare: three ways to break it'
date = 2026-11-22T12:00:00+01:00
draft = false
tags = ['vaultwarden', 'podman', 'nginx', 'cloudflare', 'self-hosting']
summary = 'A WebSocket that never upgraded, a Cloudflare default that rewrote the web vault, and a security header that made a correct password look wrong.'
ShowToc = true
+++

Vaultwarden is a lightweight, self-hosted server compatible with the
Bitwarden clients. Getting it running took me five minutes. Getting it to
actually work through my reverse proxy and Cloudflare took the rest of the
evening, and the next morning.

None of the problems were in Vaultwarden itself. All three came from layers
in front of it, each doing something reasonable on its own. This is the story
of how I found them, in case you're staring at the same symptoms.

## The setup

Vaultwarden runs as a Podman container on a pair of Debian hosts managed by
keepalived: whichever node holds the virtual IP runs the service, with its
data on a replicated volume. With Podman's **Quadlet**, the container is
described in a small file, and systemd turns it into a regular service:

```ini
# /etc/containers/systemd/vaultwarden.container
[Unit]
Description=Vaultwarden password manager
RequiresMountsFor=/srv/vaultwarden

[Container]
Image=docker.io/vaultwarden/server:1.37.5
Volume=/srv/vaultwarden:/data:Z
PublishPort=10.0.0.10:8222:80
Environment=DOMAIN=https://vault.example.com
Environment=SIGNUPS_ALLOWED=false

[Service]
Restart=on-failure
```

There's no `[Install]` section on purpose: keepalived's notify script starts
the service on the active node only. The container listens on the virtual IP,
because nginx doesn't run on these hosts: it runs on my routers, which proxy
to the backend over a private network. Cloudflare sits in front of nginx.

So a request travels through four layers: **browser → Cloudflare → nginx →
Vaultwarden**. Keep that chain in mind; every problem below lived in a
different link.

## Problem 1: the WebSocket that never upgraded

The first symptom was a client that logged in fine, then hung. Vaultwarden's
own log showed the login succeeding, followed by this:

```
GET /notifications/hub?access_token=... => 404 Not Found
```

That's the client opening a **WebSocket** for live sync. A WebSocket starts as
a normal HTTP request carrying `Upgrade` and `Connection` headers, and nginx
**does not forward those by default**. Vaultwarden received a plain `GET`,
treated it as a request for a static file, and answered 404.

The fix is the standard WebSocket block in the `location` that proxies to
Vaultwarden:

```nginx
location / {
	proxy_pass http://vaultwarden;
	proxy_http_version 1.1;
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection $connection_upgrade;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
}
```

plus a `map` at the `http` level, so ordinary requests aren't told to upgrade:

```nginx
map $http_upgrade $connection_upgrade {
	default upgrade;
	''      close;
}
```

### Testing one layer at a time

What finally made this kind of problem tractable was testing each link of
the chain with the same request, a WebSocket handshake made with `curl`:

```sh
curl -si -N --http1.1 --max-time 5 \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' \
  http://10.0.0.10:8222/notifications/hub | head -1
```

A **401 Unauthorized** is the answer you want: Vaultwarden recognized the
WebSocket and only rejected it for lacking a token. Run it against the
backend directly, then through nginx, then through the public hostname, and
the first layer that turns 401 into 404 is your culprit.

Two traps while doing this: force `--http1.1`, since WebSockets don't use the
`Upgrade` mechanism over HTTP/2, and don't silence curl with `-s` while
debugging. My first "empty" results turned out to be curl quietly failing
to verify a certificate.

## Problem 2: a Cloudflare default that rewrote the web vault

With WebSockets fixed, the web vault still misbehaved, but only through
Cloudflare. Pointing my machine straight at nginx, everything worked.

Cloudflare's security events for the hostname were empty, and I had no cache
rules. The culprit was a feature that doesn't block anything, so it never
shows up as an event: **Email Address Obfuscation**. It's **on by default**
for new zones, and it rewrites anything that looks like an email address in
HTML passing through Cloudflare, injecting a script to decode it in the
browser. A single-page application that handles your login email does not
enjoy that.

Turning it off fixed the web vault immediately. You can do it zone-wide, or,
if other sites benefit from it, with a Configuration Rule that disables it
for the Vaultwarden hostname only. You can also check it through the API:

```sh
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE/settings/email_obfuscation" \
  | jq -r '.result.value'
```

(The token needs **Zone Settings: Read**, which a DNS-only token won't have.)

## Problem 3: "Invalid master password", with the right password

This one was the most misleading. I registered an account, logged in, and
the web vault told me my master password was invalid. I recreated the
database and registered again. Same result.

Vaultwarden's log told a different story:

```
POST /identity/accounts/register/finish => 200 OK
POST /identity/accounts/prelogin/password => 200 OK
POST /identity/connect/token => 200 OK
```

Every login **succeeded on the server**. The password was right. The error
came from the browser, in the step that happens after login: decrypting the
vault, entirely client-side.

Current Bitwarden clients do that cryptography in **WebAssembly**, and
browsers only run WebAssembly when the page's Content Security Policy allows
it, with `'wasm-unsafe-eval'`. Vaultwarden sends its own CSP, which includes
it. But my nginx configuration added a site-wide CSP of its own:

```nginx
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
```

When a response carries two CSP headers, browsers enforce **both**, so the
stricter combination wins. Mine didn't allow WebAssembly, the decryption code
never ran, and the web vault reported the failure as a wrong password.

Removing nginx's CSP for that one hostname fixed it, and a `map` keeps the
policy in one place for every other site:

```nginx
map $host $csp {
	vault.example.com "";
	default "default-src 'self' http: https: data: blob: 'unsafe-inline'";
}

add_header Content-Security-Policy $csp always;
```

nginx skips `add_header` when the value is empty, so Vaultwarden's own policy
is the only one the browser sees. If you take this route, remember nginx's
inheritance rule: a server block with **any** `add_header` of its own ignores
all the ones defined at the `http` level, so the Vaultwarden block should
have none.

A quick check shows exactly what reaches the browser:

```sh
curl -sI https://vault.example.com/ | grep -i content-security
```

There should be exactly one policy, and it should contain `'wasm-unsafe-eval'`.

## A side finding: the missing intermediate

While testing nginx directly, curl refused to verify my certificate. nginx was
serving `cert.pem`, the bare certificate, instead of `fullchain.pem`, which
includes Let's Encrypt's intermediate. Browsers fill in the gap silently, and
Cloudflare doesn't mind, so it had gone unnoticed for years. It had nothing
to do with Vaultwarden, but if you've never checked:

```sh
openssl s_client -connect localhost:443 -servername vault.example.com -showcerts </dev/null 2>/dev/null \
  | grep -c ' s:'
```

One means the chain is incomplete.

## Takeaways

- **Test the chain one link at a time.** The same request, sent to each
  layer, tells you exactly where it breaks.
- **Read the server's logs before believing the client's error.** "Invalid
  master password" was a client-side failure behind a successful login.
- **Defaults are configuration too.** Cloudflare's Email Obfuscation was on
  without me ever choosing it.
- **Security headers stack.** A second CSP doesn't replace the application's
  own; it restricts it further. Let applications that ship their own policy
  keep it.
