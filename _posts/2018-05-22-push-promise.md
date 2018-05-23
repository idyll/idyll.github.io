---
layout: post
title:  "PUSH_PROMISE with Cowboy"
subtitle: "Getting Cowboy/Phoenix to Push files with H2"
date:   2017-09-09 18:00:00
categories: phoenix elixir tls
author: mark
image: /images/posts/cert-keychain.png
---

The recent release of Cowboy 2.4.0 fixes the last bug I am aware of in its H2 implementation. When Phoenix 1.4 is
released it will include alpha support for Cowboy 2 and thus HTTP/2 (or H2 for short).

I've been testing it for a while and one of the hardest things to get working was push (aka PUSH_PROMISE) support in H2.

Server push or push promise is when the server sends a client instructions to download files it knows the client will
need. The client can then receive the files before parsing the page so that they are already available in the cache.

Push has been supported by Cowboy for a while but I couldn't get it working. I tried a lot of different stuff (thats a
whole other blog post) and finally I reached out to Jake Archibald from Google who pointed out that Chrome likely wasn't
trusting my "snake oil" certificate.

It turns out that even if you tell Chrome to accept your cert, you need to trust a cert with proper SAN support to
get PUSH_PROMISE to work - *even when working off localhost*.

**Simply put, you need to use SAN or push doesn't work.**

So I've put together instructions on how to setup your own self-signed certs with SAN support.

### Self-Signed Certs with SAN using OpenSSL

Before you get started, I want to point out that I am using `test.phx.sh` as my DNS name for my app. `phx.sh`, `*.phx.sh`
and `*.dev.phx.sh` have all be setup to point to 127.0.0.1 to simplify development when using certs. So you can just
change `test.phx.sh` to `your-app.phx.sh` and it should just work for your app.

You need to setup a config file for openssl as it needs the config file to read in the SAN.
Here's an example, let's call it `phx.sh.cnf`:

{% highlight sh %}
[req]
default_bits = 4096
prompt = no
default_md = sha256
x509_extensions = v3_req
distinguished_name = dn

[dn]
C = EX
ST = Elixir
L = Phoenix
O = App
emailAddress = email@phx.sh
CN = test.phx.sh

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = test.phx.sh
DNS.2 = *.phx.sh
{% endhighlight %}

*I'd like to point out that your OS probably has a very specific openssl.cnf file that you should base any production
certs on. On macOS you will find it in `/System/Library/OpenSSL` and it contains pretty specific config that you
should be using. This is only intended for your dev environment.*

With that file in place you can simply run openssl to generate a key and cert in one swoop.

{% highlight sh %}
openssl req -new -x509 -newkey rsa:2048 -sha256 -nodes -keyout phx.sh.key -days 3560 -out phx.sh.crt -config phx.sh.cnf
{% endhighlight %}

You can use the *key* and *crt* in your Cowboy config and this gets you started.

The next step is marking the cert as trusted. Double-clicking on the `.crt` file should open *Keychain Access* where you
can mark the certificate as trusted. Open it up and there's a dropdown where you can change the trust settings. (See
above of for a picture of it.)

This will mark the cert as trusted for Safari. You may still get a warning in Chrome but PUSH_PROMISE will now work.
