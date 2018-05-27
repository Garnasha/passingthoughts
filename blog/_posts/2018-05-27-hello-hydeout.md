---
layout: post
title: Hello Hydeout
---

Today I switched to using [Hydeout](https://github.com/fongandrew/hydeout),
which involved some setup. Here are the essentials: 

### Gemfile

The README mentions changing your Gemfile, but that's for hosting locally, in
which case, their instructions are the ones you need. However, if you are hosted
on GitHub, you should instead put `remote-theme:fongandrew/hydeout` in your
\_config.yml

### \_config.yml

I just copied most of what was in hydeout/\_config.yml to my own. Some
explanation as to what some options do:

`title:` gives the title of your blog, which will be the default title of the
index page. Appears in the side bar, as well as in the human-readable name of
any links to pages using the default title (which is displayed in e.g. tab
headers).

`tagline:` is given as a link subtitle to any pages that don't provide their own.
Appears after the title of the page in the human-readable name of the link to
the page.

`description:` sidebar subtitle of your blog. Appears under the title of your
blog in the sidebar.

`paginate: 5` activates pagination and sets posts per page to 5.
`paginate_path:` lets you restrict pagination to a directory within your
project, the rest being rendered without pagination.
