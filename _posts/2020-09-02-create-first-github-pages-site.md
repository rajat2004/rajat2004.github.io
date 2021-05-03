---
title: Creating your first GitHub Pages site
date:   2020-09-02 01:27:02 +0530
---

Well, this is my first time creating a personal site using GitHub Pages, and so for the first post, why not write one regarding that :slightly_smiling_face: . There's already a lot of awesome articles and guides out there on how to go about creating a github.io site, so not going to repeat all that.
This is just going to be a short post pointing to docs and articles used, and few notes about some problems I had on the way & what could have been done better.

How to get started? Well, there's nothing better than the [official GitHub documentation](https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site), that clearly lays out the steps with nice photos on how to create a repo from scratch. If you would like to have a look at more expansive overview of Git, GitHub pages, Jekyll, etc. [here's](http://jmcglone.com/guides/github-pages/) a nice long post.

The next thing would you probably want to try out is customizing the look by changing the theme. Again see [GitHub documentation](https://docs.github.com/en/github/working-with-github-pages/adding-a-theme-to-your-github-pages-site-with-the-theme-chooser), choose a theme and it'll automatically create a `_config.yml` & set the theme.

This is where things get more hands-on. Editing themes directly from browser is fine, but for further changes such as writing posts (like this!), adding social links, etc. it's preferable to test things locally and see how the site looks before committing. So let's setup the local environment to test the site.

Some more things are required other than just the [GitHub documentation](https://docs.github.com/en/github/working-with-github-pages/setting-up-a-github-pages-site-with-jekyll), but it's the main page to follow along to ensure the correct versions of libraries are being used. First thing required is installing Jekyll on you system, for which follow [Jekyll installation](https://jekyllrb.com/docs/installation/). I'm on Ubuntu 18.04, so followed the steps for [Ubuntu here](https://jekyllrb.com/docs/installation/ubuntu/).

Now that Jekyll is installed, let's try to build and serve our site locally. The [GitHub docs](https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll) give instructions to create a new repo, etc but we already have that, so can just skip the first 5-6 steps and clone our already existing repository, `cd` to the location and ensure we're on the `master` branch, or `gh-pages` if using that.

Step 7 in the [GitHub page](https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll) indicates that we need to use the Jekyll version used by GitHub, the versions are listed [here](https://pages.github.com/versions/). At the time of writing, GitHub uses Jekyll 3.9.0, however `gem` installed 4.1.1 (check using `jekyll -v`) . So to install the correct version, first remove Jekyll using `gem uninstall jekyll` and then install the correct one using `gem install jekyll -v 3.9.0`.

Finally, we can create a Jekyll site using `jekyll new .` from the root directory of the repository. This will create the `_config.yml`, `Gemfile`, etc. files. Now edit them following Steps 8,9 in the [doc](https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll)

To build and serve the site locally, use `bundle exec jekyll serve`, and open `localhost:4000` in your browser to see the live rendered site. Any changes in the source files such as *.md files will the rendered instantly, however changes to `_config.yml` requires restarting the server. If everything looks okay, add all the files using `git add .`, write a meaningful commit message and push. Yay, we have a functional GitHub pages website with local Jekyll testing!

One option I found quite useful is `bundle exec jekyll serve --livereload --open-url`, that'll open the live website directly in the default browser when local server starts running, no need to manually open.

I'm currently using [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) as the theme, just wanted to add a note here about a mistake I made, which cost me 1-2 hours on-and-off. Follow the [Quick-Start Guide](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/) on how to add the plugin, however I did not look at the page in the beginning and was following some other article due to which I missed a step. The problem was that I didn't replace the `index.md` file already present with an `index.html` as mentioned, due to which the main page didn't get updated with any changes I made at all.

So make sure to do that, and follow the detailed [Configuration](https://mmistakes.github.io/minimal-mistakes/docs/configuration/) and further pages to setup the site to your liking! I did want to use the [`dark` skin](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#dark-skin-dark) :heart:, however the GitHub icon just goes invisible in that, and [Font Awesome](https://fontawesome.com) currently doesn't seem to have a light GitHub icon, which is used for icons in Minimal Mistakes. I did try to use a custom image rather than Font Awesome, but currently doesn't seem to be possible, at least not without some heavy modifications to the HTML, JS files. Will post any update if something works out.

Until then, happy website building!

Update: Back after a long time, changed to `dark` skin anyways.
