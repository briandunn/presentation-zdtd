# Zero Downtime Phoenix Deploys

https://github.com/briandunn

<img src="https://hashrocket.com/hashrocket_logo.svg" />

Note: Our first elixir project!
---

## Zero Downtime Deploy:

Change the behavior of the application without disrupting service.

Note: Lots of ways to achieve this. Most focus on infrastructure

---

## Infrastructure

<img height=200 src="https://upload.wikimedia.org/wikipedia/commons/thumb/5/5c/AWS_Simple_Icons_AWS_Cloud.svg/2000px-AWS_Simple_Icons_AWS_Cloud.svg.png" />

Note: blue green deploys with DNS switching
---

## Five Nines ™️

---

<img src="https://upload.wikimedia.org/wikipedia/commons/e/eb/City%2C_telephone_room_%282898490491%29.jpg" />

Note: We have a high efficiency environment with the EVM and it has tools for this. Let's use them
---

* Deploying
* Managing Configuration
* Hot Upgrades
<!-- * Live Example? -->
<!-- * Conclusions -->


Note: Now that we have defined terms, I'd like to talk about my experience with using these features.

---

# Deploying

* Distillery
* Edeliver

(nginx)

---

## Distillery

> ...takes your Mix project and produces an Erlang/OTP release, a distilled form of your raw application's components; a single package which can be deployed anywhere, independently of an Erlang/Elixir installation. No dependencies, no hassle.

Note: No hassle is relative. Some tweaks necessary to get live upgrade static assets working with phoenix
---

## Edeliver

> build and deploy Elixir and Erlang applications and perform hot-code upgrades.


---

```sh
# .deliver/config

APP="myapp"

BUILD_HOST="my-build-server.myapp.com"
BUILD_USER="builder"
BUILD_AT="/tmp/edeliver/myapp/builds"

STAGING_HOSTS="stage.myapp.com"
STAGING_USER="web"

PRODUCTION_HOSTS="www.myapp.com"
PRODUCTION_USER="web"
```

Note: This is an excerpt. Use hooks to build assets and configure
---

* build
* stage
* production

Note: can have multiple nodes in stage and production

---

build === stage === production

Note: Distillery releases are not cross platform. You can't build it on your mac.
---

Configure edeliver to build your static assets too

```sh
pre_erlang_clean_compile() {
  status "Build js"
  __sync_remote "
    set -e
    cd '$BUILD_AT'
    APP='$APP' MIX_ENV='$TARGET_MIX_ENV' $MIX_CMD build_js $SILENCE
  "
}
```

---

    mix edeliver build release

* shells in to the build host
* runs `mix release` and hooks
* copy the resulting `.tar.gz` down to your dev machine

Note: live example?
---

    mix edeliver deploy release to staging
    
* Copies the release to the staging server
* Shells in and decompresses it

---

    mix edeliver start staging
    
* Shells in and in and runs `bin/myapp start`

Note: distillery gives us start and much more, which we will see later
---

# 🍾

Ready for production?

Note: so far so good. But this is where things get a little tricky.
---

## A working strategy for configuring distillery builds in multiple environments

---

* Put *all* configuration in `sys.config`
* Build releases against a dummy example version
* Keep actual `sys.config`s on staging and production host boxes
* SymLink to the host `sys.config` on deploy

---

what is `sys.config`?

---

# Remember 12 Factor?

https://12factor.net/config
> The twelve-factor app stores config in environment variables

---

<img src="https://www.blogcdn.com/www.engadget.com/media/2007/07/7-17-07-computer_out_window.jpg" />

Note: throw that out the window. We're going old school.
---

<img src="https://dl.dropboxusercontent.com/s/p2znyi61jsuwtjn/Screenshot%202017-12-12%2017.28.36.png" />

---

what is `sys.config`?

http://erlang.org/doc/man/config.html
> A configuration file contains values for configuration parameters for the applications in the system.

It's an Erlang flat file like

```erlang
[{Application1, [{Par11, Val11}, ...]},
 ...
 {ApplicationN, [{ParN1, ValN1}, ...]}].
```

Note: small silver lining: it's typed.
---

## Can be generated from `mix.config`

```elixir
config = Mix.Config.read!("config/config.exs")
:io_lib.format('~p.~n', [config])
|> List.to_string()
|> IO.puts()
```

---

# *All* configuration

## Never do this:

```elixir
defmodule HTTPClient do
  @url Application.get_env(:my_app, :url)
end
```

Frozen at compile time
---

# Build against a dummy version

```sh
# .deliver/config
pre_erlang_get_and_update_deps() {
  status "Copy example config"
  scp config/example.secret.exs $BUILD_USER@$BUILD_HOST:$BUILD_AT/config/prod.secret.exs
}
```

Note: that way there are no secrets in your release tar files
---

```sh
# .deliver/config
LINK_SYS_CONFIG="/home/hashrocket/$APP.config"
```

```elixir
# rel/config.exs
environment :prod do
  # ...
  plugin(Releases.Plugin.LinkConfig)
end
```

---

# production ready!

* no memoized configuration
* no environment variable access
* each environment has their specific `sys.configs`

Note: production box has no node.js. No elixir. No EVM. Just nginx.
---

```sh
mix edeliver deploy production
```

---
🥂 🎉💰
---

# Hot Upgrades

---

## about live code updating in the EVM

http://erlang.org/doc/man/appup.html
> defines how an application is upgraded or downgraded in a running system.

A series of instructions on how to handle the reloading of each module

* Distillery can make one for you
* edeliver can deliver it for you

---

# We need versions!

```sh
# .deliver/config
AUTO_VERSION=build-date+commit-count+git-revision
```

```sh
ls .deliver/releases
my_app_0.0.1+20171116-381-eddcacc.release.tar.gz
```

---

# Let's build it

```sh
mix edeliver version

---> 0.0.1+20171116-381-eddcacc
```

```sh
mix edeliver build upgrade --with=0.0.1+20171116-381-eddcacc
```

* Copies that release to the build server
* Builds the latest version of master
* Generates an upgrade release
* Copies that down to your dev machine

---

# Deploy it

```sh
ls .deliver/releases
my_app_0.0.1+20171116-381-eddcacc.release.tar.gz
my_app_0.0.1+20171204-392-49ab6e0.upgrade.tar.gz
```

```sh
mix edeliver deploy upgrade to production
```

---

# 🤸

---

# What about my static assets?!

---


```elixir
# rel/config.exs
environment :prod do
  # ...
  set(post_upgrade_hook: "rel/hooks/post_upgrade")
end
```

---

```sh
# rel/hooks/post_upgrade
echo "Clearing Phoenix static asset cache"
$SCRIPT eval "'Elixir.Phoenix.Config':clear_cache('Elixir.TeamsnapStoreWeb.Endpoint')."
echo "Warming up the endpoint"
$SCRIPT eval "'Elixir.Phoenix.Endpoint.Supervisor':warmup('Elixir.TeamsnapStoreWeb.Endpoint')."
```

Note: this hook is not documented
---

## Only the from hooks fire when an upgrade is released

---

# Configuration changes

---

```sh
bin/my_app reload_config
```

Restart processes that cache configuration

```sh
bin/my_app attach
```

```elixir
iex> Application.stop(:logger) && Application.start(:logger)
```

Note: DONOT exit attach with CTRL-C. Use CTRL-D.
---
# ☠️

Note: depending on your supervision strategy this may cause thrashing / log flooding
---

# Conclusion

## To hot deploy with edeliver:

* learn to love `sys.config`

---

# thanks!

<img src="https://hashrocket.com/hashrocket_logo.svg" />

<img src="https://p6.zdassets.com/hc/settings_assets/494683/200066510/gHYqkAWYHfzU3SCj3ZbGvQ-UPLmNaXpx5ksDV2t3Ga8Uw-TeamSnap_Logo1024x225.png" />

Note: thanks hashrocket, thanks teamsnap, thanks Chicago elixir!
---
