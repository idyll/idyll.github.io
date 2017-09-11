---
layout: post
title:  "TLS for Phoenix"
subtitle: "How to setup and lockdown Phoenix TLS config."
date:   2017-09-09 18:00:00
categories: phoenix elixir tls
---

HTTP/2 support already available in Phoenix thanks to Cowboy2. Depending on how your load balancing was setup you may
now be in a situation where you now need to termiate TLS directly on Cowboy.

When I looked into it there didn't seem to be a lot of information around it so I decided to assemble the config
and details that I used here.

There's two things that I wanted to do when I set this up:

  1. only use secure tls versions and protocols.
  2. keep the keys encrypted.

My suggestions here are based on the following:

  - [https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices)
  - [http://ezgr.net/increasing-security-erlang-ssl-cowboy/](http://ezgr.net/increasing-security-erlang-ssl-cowboy/)
  - [https://ninenines.eu/docs/en/ranch/1.3/manual/ranch_ssl/](https://ninenines.eu/docs/en/ranch/1.3/manual/ranch_ssl/)

One challenge that we have is the DH params group file. I'd prefer to generate this on the server but it
takes too long to be practical. I have excluded it for the time being as it makes not sense to generate it
but then store it in a git repos.

However, your deployment process may be different so if you can take advatage of it, generate it as follows:
{% highlight sh %}
openssl dhparam -out dh-params.pem 2048
{% endhighlight %}

Beyond that we are encrypting our keyfile and then only provide the password to the production ENV.
The resulting config looks as follows:

{% highlight elixir linenos %}
config :phoenix, YourApp.Endpoint,
  https: [
           port: 443,
           host: "yourapp.example.com",
           cacertfile: "/path/to/certs/cacert.pem",
           certfile: "/path/to/certs/cert.pem",
           keyfile: "/path/to/certs/key.pem",
           password: "keyfile-password-so-read-this-from-the-env",
           versions: [:"tlsv1.2"],
           dhfile: "/path/to/certs/dh-params.pem",
           ciphers: [
                      "ECDHE-ECDSA-AES128-GCM-SHA256", "ECDHE-ECDSA-AES256-GCM-SHA384", "ECDHE-ECDSA-AES128-SHA",
                      "ECDHE-ECDSA-AES256-SHA", "ECDHE-ECDSA-AES128-SHA256", "ECDHE-ECDSA-AES256-SHA384",
                      "ECDHE-RSA-AES128-GCM-SHA256", "ECDHE-RSA-AES256-GCM-SHA384", "ECDHE-RSA-AES128-SHA",
                      "ECDHE-RSA-AES256-SHA", "ECDHE-RSA-AES128-SHA256", "ECDHE-RSA-AES256-SHA384",
                      "DHE-RSA-AES128-GCM-SHA256", "DHE-RSA-AES256-GCM-SHA384", "DHE-RSA-AES128-SHA",
                      "DHE-RSA-AES256-SHA", "DHE-RSA-AES128-SHA256", "DHE-RSA-AES256-SHA256"
                    ],
           secure_renegotiate: true,
           reuse_sessions: true,
           honor_cipher_order: true,
           max_connections: :infinity
        ]
{% endhighlight %}

In my case, we use distillery and read much of our configuration from the ENV. And we are not generating the DHPARAMS
So our file looks more like this:

{% highlight elixir linenos %}
config :phoenix, YourApp.Endpoint,
  https: [
           port: "${PHX_TLS_PORT}",
           host: "${PHX_HOST}",
           cacertfile: "${CA_FILE_PATH}",
           certfile: "${CERTFILE_PATH}",
           keyfile: "${KEYFILE_PATH}",
           password: "${KEYFILE_PASSWORD}",
           versions: [:"tlsv1.2"],
           ciphers: [
                      "ECDHE-ECDSA-AES128-GCM-SHA256", "ECDHE-ECDSA-AES256-GCM-SHA384", "ECDHE-ECDSA-AES128-SHA",
                      "ECDHE-ECDSA-AES256-SHA", "ECDHE-ECDSA-AES128-SHA256", "ECDHE-ECDSA-AES256-SHA384",
                      "ECDHE-RSA-AES128-GCM-SHA256", "ECDHE-RSA-AES256-GCM-SHA384", "ECDHE-RSA-AES128-SHA",
                      "ECDHE-RSA-AES256-SHA", "ECDHE-RSA-AES128-SHA256", "ECDHE-RSA-AES256-SHA384",
                      "DHE-RSA-AES128-GCM-SHA256", "DHE-RSA-AES256-GCM-SHA384", "DHE-RSA-AES128-SHA",
                      "DHE-RSA-AES256-SHA", "DHE-RSA-AES128-SHA256", "DHE-RSA-AES256-SHA256"
                    ],
           secure_renegotiate: true,
           reuse_sessions: true,
           honor_cipher_order: true,
           max_connections: :infinity
        ]
{% endhighlight %}

Note that we are setting files paths dynamically which allows us to switch between certificate sets between staging
and production ENVs.
