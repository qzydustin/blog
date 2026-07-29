+++
title = "Setting Up a Jekyll Blog"
date = 2022-07-06T16:29:28-07:00

[taxonomies]
categories = ["DevOps"]
tags = ["blogging", "tooling", "ruby"]
+++

Jekyll is a static site generator written in Ruby. It renders Markdown through Liquid templates into static HTML. The published site is a directory of files with no database and no server-side runtime at request time.

On Apple Silicon, the Ruby toolchain needs a little setup first: Homebrew, a Ruby version manager, and the gem path.

<!--more-->

## The Ruby toolchain on Apple Silicon

macOS ships a system Ruby, but it is old and Apple has deprecated it. Install a separate Ruby through Homebrew.

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

On Apple Silicon, Homebrew installs under `/opt/homebrew`, not the Intel `/usr/local`. The PATH setup below assumes that. Install `chruby` and `ruby-install`, then pull a current Ruby.

```shell
brew install chruby ruby-install
ruby-install ruby
```

`chruby` switches between Ruby versions. `ruby-install` fetches and builds them. Wire the Homebrew Ruby into your shell so gems resolve.

```shell
echo "\n# Ruby" >> ~/.zshrc
echo "if [ -d \"/opt/homebrew/opt/ruby/bin\" ]; then" >> ~/.zshrc
echo "  export PATH=\"/opt/homebrew/opt/ruby/bin:\$PATH\"" >> ~/.zshrc
echo "  export PATH=\"\$(gem environment gemdir)/bin:\$PATH\"" >> ~/.zshrc
echo "fi" >> ~/.zshrc
```

Restart your terminal, then verify. `chruby` lists installed versions. `ruby -v` reports the one in use.

## Jekyll and the bundle

With Ruby in place, install Jekyll and bundler.

```shell
gem install jekyll bundler
```

Bundler reads a `Gemfile`, resolves gem versions, and records them in `Gemfile.lock`. Running through `bundle exec` uses that lock, so an upstream gem release cannot silently change the rendered site. Without it, two machines with the same Jekyll version can still diverge on a transitive dependency.

Scaffold the site.

```shell
jekyll new blogname
bundle add webrick
```

`webrick` is the Ruby standard library HTTP server. It was demoted out of the default gem set in Ruby 3.0, so Jekyll's dev server needs it added explicitly or `jekyll serve` fails to load.

## The site convention

A fresh Jekyll site has a fixed shape. `_config.yml` holds site-wide settings: title, URL, plugins, build flags. Posts live in `_posts/`, one file per entry, named `YYYY-MM-DD-title.md`. The date in that filename becomes the post's date and feeds the permalink, so renaming a published post moves its URL. `_layouts/` and `_includes/` hold the Liquid templates that wrap content. Liquid is the templating language: layouts loop over posts, interpolate variables, and compose partials at build time.

Serve it.

```shell
bundle exec jekyll serve
```

The site builds into `_site/`, rebuilds on every file change, and serves at `http://localhost:4000`.

## Choosing Jekyll or Hugo

Hugo is a single Go binary with no runtime dependencies and builds sites in milliseconds. This blog now uses Hugo. Jekyll remains convenient on GitHub Pages, which builds it on push without requiring separate CI. Choose between a managed Ruby environment and a self-contained binary.
