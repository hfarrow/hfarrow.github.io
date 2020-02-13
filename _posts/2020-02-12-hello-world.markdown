---
title:  "Hello, World!"
date:   2020-02-12 23:21:00 -0800
categories: meta
---
This post will describe the steps taken to setup this site. [GitHub Pages](https://pages.github.com/) hosts the site and it is powered by the static site generator [Jekyll](https://jekyllrb.com/). The domain [https://heathfarrow.dev]() is managed through [Google Domains](https://domains.google.com). I modify the site locally on Ubuntu with a text editor by cloning the site from [hfarrow/hfarrow.github.io](https://github.com/hfarrow/hfarrow.github.io) and running it locally via `bundle exec jekyll _3.8.5_ serve`. The site is published by pushing all changes back to GitHub where GitHub Pages will build and deploy the site.

Generally speaking, getting everything setup and running was as simple as following the [GitHub Pages Guide](https://help.github.com/en/github/working-with-github-pages). At first there were some parts I didn't understand immediately, especially considering I'm not familiar with Ruby, Jekyll, or GitHub Pages.

## Issues
One issue was encountered while installing Ruby gems locally for testing the site. HTTP requests were timing out or failing to resolve. After searching for the errors I was seeing, I found a user suggesting that IPv6 was the problem. Instead of trying to fix a local networking issue I decided to disable IPv6 on my system.

~~~ sh
sudu vim /etc/sysctl.conf

# Add the following lines
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Make the changes active
sudo sysctl -p

# Verify the following command outputs '1' to indicate
# IPv6 is disabled
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
~~~

## Customization
After searching various sites for free Jekyll themes, I landed on the popular theme [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/). Once again I followed documentation to configure the site to my liking which can be viewed in the sites [source repo](https://github.com/hfarrow/hfarrow.github.io).

## Continued Learning
Jekyll uses a markup language named [kramdown](https://kramdown.gettalong.org/index.html). While I have used markup
languages in past, that usage has not been very heavy. Looking through the docs for both kramdown and Miminal Mistakes
I noticed lots of useful functionality that will require some time to memorize or be aware of whats possible.

I also have the option of customizing the theme directly as well. Extending, overriding, or changing it's css and much
more. I'd prefer to work with it out of the box. I've noticed the font size for the content of a page or post is larger
than I'd like. At some point, I'll want to adjust the font size.
