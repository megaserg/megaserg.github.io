# Environment setup

Install RVM as described at [rvm.io](https://rvm.io/):

    $ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
    $ \curl -sSL https://get.rvm.io | bash -s stable

Install Ruby with OpenSSL support:

    $ brew install openssl
    $ rvm install 2.4 --with-openssl-dir=/usr/local/opt

This should install the latest version and use it as default.

Install Jekyll:

    $ gem install jekyll bundler

In the blog directory, run:

    $ bundle install

Now we can start the local server (watch autoenabled) with:

    $ jekyll serve