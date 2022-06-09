---
layout: post
title: "Distillery to Releases"
subtitle: "For Phoenix Applications Running in Kubernetes"
date: 2022-06-09 22:00:00
categories: programming
author: mark
image: /images/posts/pine.jpg
---

OTP25 was released a couple of weeks ago bringing JIT support to ARM (aka Apple M1). However, Distillery
does not support OTP25. And I doubt it ever will given that it's been all but replaced by Releases.

Distillery had a good run. But now it's time to move on to releases.

This is a guide to help migrate from Distillery to Releases for Phoenix Applications built as Docker images
running on Kubernetes with clustering provided by Libcluster.

(This is a specific case because you need the `POD_IP` set as part of the name in `vm.args` in order
to cluster the nodes.)

Here's a quick overview of the changes:

1. Update `mix.exs` to remove Distillery
2. Remove the Distillery related files
3. Add new `vm.args` as `/rel/vm.args.exs`
4. Generate the release configuration
5. Update your configuration to load in the ENV
6. Update your Dockerfile

Upon deploying the application you should be able to access the remote console using:

```sh
./bin/app remote
```

## 1. Update `mix.exs` to remove Distillery

Remove the following lines from `mix.exs`:

```elixir
      # Distillery
      {:distillery, "~> 2.1"},
      # Config Providers
      {:config_tuples, "~> 0.4"},
```

## 2. Remove the Distillery related files

Remove the following files and directories:

```
/rel/config.exs
/rel/hooks/*
/rel/plugins/*
/rel/vm.args
```

## 3. Add new the old vm.args configuration in `/rel/env.sh.eex`

You can generate this file using: `mix release.init` (which is separate from the phoenix command that you 
will run in the next step).

The following is my `env.sh.eex` file:

```elixir
# rel/env.sh.eex
export RELEASE_DISTRIBUTION=name
export RELEASE_NODE="app@${POD_IP}"
export RELEASE_COOKIE="${ERLANG_COOKIE}"

```

Note that you will need to replace the app name with the name of your otp app.
And you can directly set the cookie. And `RELEASE_DISTRIBUTION` is NOT OPTIONAL.
(I messed that part up for a while and was quite confused.)

## 4. Generate the release configuration.

Run `mix phx.gen.release` to generate the files needed to build the release.

This will generate files in `/rel/overlays/bin` needed to start the server and run migrations.

You will probably want to update `lib/wildcast/release.ex` to change the behaviour of the load_app function:

```elixir
   defp load_app do
     Application.ensure_all_started(@app)
   end
```
so that you don't have errors with `extra_applications` such as `ssl` when running the migrations.

## 5. Update your configuration to load in the ENV

The default config provider in Distillery allows you to load in the ENV from special strings
that are formatted like: `"${ENV_VAR}"`.

This are replaced with the ENV variables at runtime.

Instead you will need to do move ALL RUNTIME config to a file called `releases.exs` and load in the
ENV VARS using either `System.fetch_env!/1` or `System.get_env/2`.

Use `System.fetch_env!/1` when you want the app to crash if the proper ENV is not set.

Use `System.get_env/2` when you want to use a default value if the ENV is not set OR if you want to
allow `nil` for certain settings.

For example:

```
config :app, App.Repo,
   url: System.fetch_env!("DATABASE_URL"),
   pool_size: String.to_integer(System.get_env("DATABASE_POOL_SIZE", "20"))
```

requires that `DATABASE_URL` is set in the ENV but allows `DATABASE_POOL_SIZE` to be `nil`.

## 6. Update your Dockerfile

Finally, update your Dockerfile to use the the OTP25 images and to trigger migrations before starting the server.

You can also remove `REPLACE_OS_VARS` from the dockerfile if you were setting it there. 

```dockerfile
FROM elixir:1.13-otp-25-alpine as build
.
.
.
RUN MIX_ENV=prod mix release
.
.
.
FROM erlang:25-alpine
.
.
.
CMD ["sh", "-c", "/opt/app/bin/migrate && /opt/app/bin/app start"]
```


