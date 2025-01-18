# [sergey-melnychuk.github.io](https://sergey-melnychuk.github.io)
Blog on software engineering

```bash
## Install rvm (https://rvm.io/)
curl -sSL https://get.rvm.io | bash -s stable

# https://github.com/rbenv/ruby-build/discussions/2053#discussioncomment-6725967
rvm install 3.1.2 --with-openssl-dir=$(brew --prefix openssl@3)

source "$HOME/.rvm/scripts/rvm"
rvm 3.1.2 --default

gem install bundler
bundle install

## Run locally
bundle exec jekyll serve --host=0.0.0.0
```
