---
title: "Blogging on GitHub Pages"
layout: post
excerpt: "How to start a blog on GitHub Pages. "
tags:
  - blogging
  - github-pages
---

## About  
<https://help.github.com/en/github/working-with-github-pages/about-github-pages#about-github-pages>
> GitHub Pages is a static site hosting service that takes HTML, CSS, and JavaScript files straight from a repository on GitHub, optionally runs the files through a build process, and publishes a website. You can see examples of GitHub Pages sites in the GitHub Pages examples collection.  
You can host your site on GitHub's github.io domain or your own custom domain.

## Getting started
* Decide on a static site generator. Reference: <https://www.staticgen.com/>  
* Choose a template. Examples -  [Jekyll](https://jekyllthemes.io/free), [Hugo](https://themes.gohugo.io/), [Pelican](http://www.pelicanthemes.com/), [GatsbyJS](https://www.gatsbyjs.org/starters/)  
* I chose [Hydeout](https://fongandrew.github.io/hydeout/) as:
  * GitHub Pages supports building Jekyll sites natively.
  * I wanted a simple website that is [designed to last](https://jeffhuang.com/designed_to_last/). [Hyde](https://hyde.getpoole.com/) fits the bill.
  * Hydeout is the still maintained fork of Hyde with some neat added features.
* Fork the repo to your GitHub account and change the repo name under settings to - \<github_username\>.github.io. This will be your blog's URL.
* Now we can clone the repo locally and customise its looks as we desire.
* Add posts under the `_posts` directory in [Markdown](https://guides.github.com/features/mastering-markdown/) format. [Guide to writing posts](https://jekyllrb.com/docs/posts/).
* Edit `_config.yml` to add your data.
* Commit and push your changes.
* Now GitHub will start building the site. You can check the status under Repository Settings > GitHub Pages.

## Writing Posts
I use VSCode with [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) extension to write posts.  

## Local hosting for Development
While I was customising the looks of the website I felt a need of local hosting as it would be faster than pushing and waiting for GitHub to process the changes.  
For this I found [docker-github-pages](https://github.com/Starefossen/docker-github-pages) to be excellent. The only issue is that you need to delete the Gemfile inside the repo to run the container. Note that without a Gemfile `bundle exec` command will not work.  

* Run powershell.
* cd inside the repository.
* Run `docker run -it --rm -v ${PWD}:/usr/src/app -p "4000:4000" starefossen/github-pages`
* Site is live at <http://127.0.0.1:4000/>.
