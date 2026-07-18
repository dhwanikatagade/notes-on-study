---
layout: default
---

# Setup github pages site for local dev

## Setup ruby
- Use RVM as a package manager for ruby versions
  - Install RVM as per instructions [here](https://rvm.io/rvm/install)
    - ```bash
      $ gpg --keyserver keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
      $ curl -sSL https://get.rvm.io | bash
      ```
  - To update RVM to the latest stable version
    - ```bash
      $ rvm get stable
      ```
- Install the ruby version that is required by github pages as a dependency
  - Ruby 3.3.4 is the current [github pages dependency version](https://pages.github.com/versions.json)
  - ```bash
    $ rvm install 3.3.4
    $ source ~/.rvm/scripts/rvm
    $ rvm use 3.3.4 --default
    ```

## Setup the ruby gems for github pages
- Install bundler to manage gem dependencies
  - ```bash
    $ gem install bundler
    ```
- Install the gems from the Gemfile
  - ```bash
    $ cd <github pages repository root directory>
    $ bundler install
    ```
  - This installs the github-pages gem version mentioned in the Gemfile in the current directory
- To update the github-pages gem
  - ```bash
    $ bundle update github-pages
    ```
  - This updates the major or minor version of the gem as per the version restriction in the Gemfile

## Running a local github pages server
- ```bash
  $ bundle exec jekyll serve
  ```


### References:
1. [Installing RVM](https://rvm.io/rvm/install)
1. [Github Pages dependencies](https://pages.github.com/versions.json)
1. [Jekyll getting started](https://jekyllrb.com/)
1. [About GitHub Pages and Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)
1. [Testing your GitHub Pages site locally with Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll)

