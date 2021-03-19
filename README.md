# My Personal Blog

You can find my personal blog [here](https://manuelmauro.github.io/blog/).

## Install Dependencies

Install Ruby and other prerequisites:

```bash
sudo apt-get install ruby-full build-essential zlib1g-dev
```

Avoid installing RubyGems packages (called gems) as the root user. Instead, set up a gem installation
directory for your user account. The following commands will add environment variables to your
 ~/.bashrc file to configure the gem installation path:

```bash
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Finally, install Jekyll and Bundler:

```bash
gem install jekyll bundler
```

That’s it! You’re ready to start using Jekyll.

## Build
Build the site and make it available on a local server.

```bash
bundle exec jekyll serve
```

Browse to `http://localhost:4000`
