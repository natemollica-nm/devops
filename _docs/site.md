---
name: Site
layout: default
---

## Install and Configure Ruby & Jekyll

The site is run using Ruby and Jekyll, so you'll need to install and configure
your environment. To install Jekyll on macOS, you need a proper Ruby development environment. 
macOS comes preinstalled with Ruby, but don’t use that version to install Jekyll.

Install [Ruby] and [Jekyll]:
* `brew install chruby ruby-install xz`
* `ruby-install ruby 3.1.3`
* Update `.zshrc`

  ```zsh
  # File: ~/.zshrc
  # run 'chruby' to see actual version
  source $(brew --prefix)/opt/chruby/share/chruby/chruby.sh
  source $(brew --prefix)/opt/chruby/share/chruby/auto.sh
  chruby ruby-3.1.3
  ```
* Verify Ruby Version:

  ```shell
  $ ruby -v
  ruby 3.1.3p185 (2022-11-24 revision 1a6b16756e) [arm64-darwin23]
  ```    

* Install latest Jekyll gem: `gem install jekyll`
* Install webrick bundle gem: `bundle add webrick`

---

## Just the Docs

The site is templated using a Jekyll site that uses the [Just the Docs] theme. 
You can easily set the created site to be published on [GitHub Pages] – the [README] 
file explains how to do that, along with other details.

If [Jekyll] is installed on your computer, you can also build and preview the created site *locally*. 
This lets you test changes before committing them, and avoids waiting for GitHub Pages.[^1] And you 
will be able to deploy your local build to a different platform than GitHub Pages.

More specifically, the created site:

- uses a gem-based approach, i.e. uses a `Gemfile` and loads the `just-the-docs` gem
- uses the [GitHub Pages / Actions workflow] to build and publish the site on GitHub Pages

Other than that, you're free to customize sites that you create with this template, however you like. You can easily change the versions of `just-the-docs` and Jekyll it uses, as well as adding further plugins.

[Browse our documentation][Just the Docs] to learn more about how to use this theme.

To get started with creating a site, simply:

1. click "[use this template]" to create a GitHub repository
2. go to Settings > Pages > Build and deployment > Source, and select GitHub Actions

If you want to maintain your docs in the `docs` directory of an existing project repo, see [Hosting your docs from an existing project repo](https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md#hosting-your-docs-from-an-existing-project-repo) in the template README.
[^1]: [It can take up to 10 minutes for changes to your site to publish after you push the changes to GitHub](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site).

----

## Building Site Locally

If you're looking to add pages or update any documentation, Jekyll allows you
to build and locally run the site as described below.

Build site locally:

```shell
$ bundle install
```

Run site locally:
```shell
$ bundle exec jekyll serve
Configuration file: /Users/username/devops/_config.yml
            Source: /Users/username/devops
       Destination: /Users/username/devops/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.389 seconds.
 Auto-regeneration: enabled for '/Users/username/devops'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
```
> Note: If you've installed Ruby 3.0 or later (which you may have if you installed the default version via Homebrew), you might get an error at this step. That's because these versions of Ruby no longer come with webrick installed.
To fix the error, try running bundle add webrick, then re-running bundle exec jekyll serve.

To preview your site, in your web browser, navigate to [http://localhost:4000](http://localhost:4000).


_ref: [Testing your GitHub Pages site locally with Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll#building-your-site-locally)_

## Site Layout

**Directories and Files**:

* **/assets**: Store images, stylesheets (CSS), and JavaScript files.
* **/_docs**: Contains the Markdown (.md) files for documentation.
* **/_runbooks**: Contains the Markdown (.md) files for operational runbooks.
* **index.md**: The landing page (home page) of the site.


[Ruby]: https://www.ruby-lang.org/en/
[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
[GitHub Pages]: https://docs.github.com/en/pages
[README]: https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
[Jekyll]: https://jekyllrb.com
[GitHub Pages / Actions workflow]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/
[use this template]: https://github.com/just-the-docs/just-the-docs-template/generate