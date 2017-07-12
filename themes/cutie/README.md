---
title: 'Hexo theme: cutie'
date: 2017-06-27 10:31:00
tags:
	- hexo
	- hexo theme
	- card
	- blogging
categories:
	- projects
photos:
  - cutie.png
icon:
  - /images/hexo.svg
---

_cutie_ is a responsive hexo theme heavily inspired by the clean and user friendly design of [www.linpx.com](http://www.linpx.com).

## Intro

### Features

* Responsive design
	* Single and dual column layout
	* Bottom navigation menu
* Configurable
	* Navigation menu (name, link and icon)
	* Disqus comment section
	* Google analytics
	* Slogan
	* Signature
* Extra
	* Mathjax
	* Instant click
	* Search page template
	* Lightbox support

### Demo

Visit my [personal website](https://qutang.github.io) for the demo.

## Installation and usage

1. clone repository into `themes` folder of your hexo website and rename the folder to `cutie`

```bash
git clone https://github.com/qutang/hexo-theme-cutie.git
```

1. A sample snippet about the theme in `_config.yml` of the website:

```yaml
# Be careful about the indent
theme: cutie

google-analytics: UA-xxxxxxxx-1

cutie:
  slogan: This is a slogan.
  signature: SIGNATURE

# Configure navigation menu, provide a relative link and a path to icon (icon should better be square)
# If menu is absent, only "archive" and "search" menu items will be preserved
# Configurable menu items should not exeed 4, otherwise the last ones will be ignored

  menu:
    Resume: 
      link: /resume/
      icon: /images/resume.svg
    Projects: 
      link: /categories/projects/
      icon: /images/projects.svg
    Notes: 
      link: /categories/notes/
      icon: /images/notes.svg
    Fun: 
      link: /categories/fun/
      icon: /images/fun.svg

# As long as the name matches the font awesome icon name, you can add even more social links
# As the tip: the social link can be a QRCODE link
  social:
    medium: 
    skype: 
    slack: 
    steam: 
    tumblr: 
    whatsapp: 
    youtube: 
    snapchat: 
    instagram: 
    qq: 
    google-plus: 
    twitter: 
    facebook: 
    weibo: 
    weixin: 
    github: https://your/github/profile/link
    linkedin: https://your/linkedin/profile/link
```

2. A set of default icons, referring using path(`/images/icon_name.svg`):
	* [archive](https://qutang.github.io/images/archive.svg)
	* [fun](https://qutang.github.io/images/fun.svg)
	* [home](https://qutang.github.io/images/home.svg)
	* [notes](https://qutang.github.io/images/notes.svg)
	* [projects](https://qutang.github.io/images/projects.svg)
	* [resume](https://qutang.github.io/images/resume.svg)
	* [search](https://qutang.github.io/images/search.svg)
	* [uncategorized](https://qutang.github.io/images/uncategorized.svg)

3. Add search page
	1. Create a new page called `search`
	1. Use layout `searching` in the front matter of `search` page

  ```yaml
  ---
  layout: search
  ---
  ```

4. It is recommended to use the hexo prism js plugin for code highlight.

5. Custom post icon

Use `icon: path/to/your/icon` in post front matter to use custom icon when displaying in home page instead of default category icon.

## Contribution
Post feature request or bugs [here](https://github.com/qutang/hexo-theme-cutie/issues), or send me pull request.

## Acknowledge

1. [www.linpx.com](http://www.linpx.com)
1. [Flaticon](http://www.flaticon.com/)
1. [InstantClick](http://instantclick.io)