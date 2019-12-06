---
title: "Blogging on Github"
layout: post
excerpt: "How to start a blog on Github using Github pages. "
tags:
  - blogging
  - github-pages
---

## About  
<https://help.github.com/en/github/working-with-github-pages/about-github-pages#about-github-pages>
> GitHub Pages is a static site hosting service that takes HTML, CSS, and JavaScript files straight from a repository on GitHub, optionally runs the files through a build process, and publishes a website. You can see examples of GitHub Pages sites in the GitHub Pages examples collection.  
You can host your site on GitHub's github.io domain or your own custom domain.

My reasons for choosing this over established blogging platforms (Wordpress, Medium) are:
* Can customize the website exactly how you want it to.
* Since posts are written in [Markdown](https://en.wikipedia.org/wiki/Markdown), it will be very easy to switch to a new service if need be (Netlify etc).

## Getting started
* Decide on a static site generator. Reference: <https://www.staticgen.com/>  
* Choose a theme. Examples -  [Jekyll](https://jekyllthemes.io/free), [Hugo](https://themes.gohugo.io/), [Pelican](http://www.pelicanthemes.com/)  
I went for [Hydeout](https://fongandrew.github.io/hydeout/) as I am not really good at Javascript and I really liked Jekyll's Hyde theme and this is the up-to-date version with some added features.
* Fork the theme repo to your Github account and change the repo name under settings to - \<github_username\>.github.io. This will be your blog's URL.
* Now we can clone the repo locally and customise its looks as we desire.
* Add posts under the `_posts` directory in [Markdown](https://guides.github.com/features/mastering-markdown/) format. [Guide to write posts](https://jekyllrb.com/docs/posts/).
* Edit `_config.yml` to add your data.
* Commit and push your changes.
* Now Github will start building the site. You can check the status under Settings > GitHub Pages.

Please feel free to use my [repo](https://github.com/amard33p/amard33p.github.io) as a base in case you liked it.

## Writing Posts
I use VSCode with [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) extension to write posts.  


## Local hosting for Development
While I was customising the looks of the website I felt a need of local hosting as it would be faster than pushing and waiting for Github to process the changes.  
For this I found [docker-github-pages](https://github.com/Starefossen/docker-github-pages) to be excellent. The only issue is that you need to delete the Gemfile inside the repo to run the container. Note that without a Gemfile bundle exec will not work.  

* Run powershell.
* cd to repository and delete the Gemfile.
* `docker run -it --rm -v ${PWD}:/usr/src/app -p "4000:4000" starefossen/github-pages`
* Browse to 127.0.0.1:4000 to see your site.
