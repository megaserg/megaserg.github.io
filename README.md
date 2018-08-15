# Environment setup

Install `rbenv` and Ruby:

    $ brew install rbenv
    $ rbenv install 2.5.1  # latest stable as of 2018-08-12
    $ rbenv global 2.5.1

Install Jekyll and Bundler:

    $ gem install jekyll bundler

In the blog directory, run:

    $ bundle install

Now we can start the local server (watch autoenabled) with:

    $ jekyll serve

For crossposting to Medium, set variables [as described here](https://github.com/aarongustafson/jekyll-crosspost-to-medium) and build:

    $ MEDIUM_INTEGRATION_TOKEN=... MEDIUM_USER_ID=... jekyll build
