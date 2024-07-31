+++
title = 'Embracing Man Pages'
date = 2024-07-11T19:49:49Z
draft = false
+++

Nowadays, the browser is always open on my computer and I spend most of my time there. If there is a problem or I need to look up the parameters of a command, the first thing I do is look it up on the web. Now I want to go back to using man pages and rediscover the romance and essence that they convey.

## Why Man Pages?

- **Always there when you need them**: Unlike the internet, which requires connectivity, man pages are always right there on your system. Whether you're on a plane, in a remote area, or just want to avoid the distractions of the web, man pages are a reliable companion.

- **Straight to the point**: Man pages don't mess around. They deliver exactly what you need: clear, concise information about commands, their options, and examples without any ads in the middle.

- **A deeper understanding**: Using man pages isn't just about finding quick answers; it's about truly understanding the system you're working with. They encourage you to think, to explore the underlying principles of Linux, and to become a more competent and confident user.

## Consuming man pages

Under **Arch Linux**, before you can use man pages, you must install them:

```bash
pacman -S man-db man-pages
```

Instead on **Debian** they are available out of the box with the installation, just use it:

```bash
man man
```

## Tricks

### Colors on man pages

To bring some happiness and have more readable man pages, you can add colours by editing your `~/.bashrc` or `~/.zshrc` and adding them:

```bash
# Man pages in Monokai-style
export LESS_TERMCAP_mb=$'\e[1;35m'
export LESS_TERMCAP_md=$'\e[1;33m'
export LESS_TERMCAP_so=$'\e[01;44;37m'
export LESS_TERMCAP_us=$'\e[1;32m'
export LESS_TERMCAP_me=$'\e[0m'
export LESS_TERMCAP_se=$'\e[0m'
export LESS_TERMCAP_ue=$'\e[0m'
export GROFF_NO_SGR=1
```

### Efficient Navigation

Man pages are displayed using the `less` pager, which has numerous powerful navigation features:

- **Search within man pages**: Press `/` followed by the search term to find specific text within the page. Use `n` to move to the next occurrence and `N` to move to the previous one.
- **Navigate by sections**: Use `g` to go to the beginning and `G` to go to the end of the document.
