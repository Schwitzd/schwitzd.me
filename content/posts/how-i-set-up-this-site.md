+++
title = 'How I Set Up This Site'
date = 2024-05-04T09:23:32Z
draft = false
+++

While creating this blog, I thought why not write my first post documenting the steps I took to make it all happen? Maybe this could help other people who have decided to take the same journey as me, so let's get started!

## Past experiences

Actually this is my second blog, for several years I maintained a website called TuxLinux where I wrote about my first experiences in the Linux world, all using [WordPress](https://wordpress.com/). But this time I was more determined to try something new (at least for me), so I started searching the internet for something like this:

* Lightweight compared to classic CMS
* Something totally nerdy
* BaC (Blog as Code, I coined a new term?!) to host on my Github

After some research I came across [Hugo framework](https://gohugo.io) and it was love at first sight.

## Architecture

Being an infrastructure guy can only mean having a dedicated page about the [architecture](/architecture) of my site with the explanations of my choices.

## Installing & Preparing Hugo

To create a [Hugo](https://gohugo.io) website, you need to install the [Hugo package](https://gohugo.io/installation/) based on your operating system. Personally, I don't like to install too much softwares on my computer and it's always a good excuse to use containers.. so I created a [Docker image](https://github.com/Schwitzd/docker-hugo) for this purpose:

{{< highlight bash >}}
# clone the repository
git clone https://github.com/Schwitzd/docker-hugo.git

# build the image
cd docker-hugo
docker build -t docker-hugo .

# Run the container in development mode
docker run --rm -it \
    -p 1313:1313 \
    -u "$(id -u):$(id -g)" \
    -v /host/site:/site docker-hugo \
    server --bind 0.0.0.0 -D
{{< / highlight >}}

I quickly realised that I was in a chicken-egg situation.. to spawn a Hugo development server, I need an existing Hugo website, but how I can create an Hugo website without the Hugo CLI?! so I decided to write a small bash script that would check if a website already exists on the mounted volume, and if not, launch a wizard to create one.

The first time you start the container in an empty folder, you will see:

```comment
Checking for Hugo website in the current directory...
No Hugo website found in the current directory.
Do you want to create a new Hugo website here? (y/n): y
Enter the name for your new Hugo website: MyWebsite
Choose the configuration format (TOML/YAML/JSON): YAML
Congratulations! Your new Hugo site was created in /site/MyWebsite.
```

Hugo gives you the choice to have the [configuration file](https://gohugo.io/getting-started/configuration) in three formats: [YAML](https://yaml.org/), [TOML](https://toml.io/en/) or [JSON](https://www.json.org/). My choice is dictated only by a cosmetic reason... [TOML](https://toml.io/en/) reminds me of the good old [INI format](https://en.wikipedia.org/wiki/INI_file), heavily used in the early years of Windows but still in use by creapy software. I love [JSON](https://www.json.org/) but in different contexts, so the winner is [YAML](https://yaml.org/)!

## Getting Started

Once the website was created, the nightmare began... choosing a theme! Hugo provides an official site [themes.gohugo.io](https://themes.gohugo.io/) where you can choose whatever you like, my chooise fall on [PaperMod](https://github.com/adityatelange/hugo-PaperMod).

Once you have followed the installation instructions for your theme and done a basic configuration of the `hugo.yaml` file, you will be able to view your website by visiting `http://localhost:1313` in a browser.

## hugo.yaml

This file is used to configure and personalise your site. This is my [hugo.yaml](https://github.com/Schwitzd), each line is explained in the [Hugo configuration docs](https://gohugo.io/getting-started/configuration).

### Syntax Highlighting

Hugo uses [Chroma](https://github.com/alecthomas/chroma) for syntax highlighting, but PaperMod theme allows to use [Highlight.js](https://github.com/highlightjs/highlight.js/). I will stay with Chroma, this is the change I did on the config:

{{< highlight yaml >}}
params:
  assets:
    disableHLJS: true

markup:
  highlight:
    codeFences: true
    guessSyntax: false
    lineNos: false
    noClasses: true
    style: monokai
{{< / highlight >}}

My favourite style theme is **Monokay** but [here](https://xyproto.github.io/splash/docs/all.html) you can see the full list of available themes.

## Pages structure

I have decided to put my **posts** in the *./content/posts* folder and my **pages** in the *./content* folder, so I need to run the following commands depending on what I want to create:

{{< highlight bash >}}
# New post
hugo new posts/my-page.md

# New article
hugo new my-page.md
{{< / highlight >}}

In the *hugo.yaml* add:

{{< highlight yaml >}}
permalinks:
    posts: /posts/:title
{{< / highlight >}}

## Search

Configuring the search functionality is straightforward, follow the [official documentation](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#search-page).
These are the Fuse.js settings i have configured:

{{< highlight yaml >}}
params:
    fuseOpts:
        ignoreLocation: true
        keys: ["title", "permalink", "summary", "content"]
{{< / highlight >}}

## Customizations

All these customisations are strictly related to the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

### Update theme

This may change depending on your theme and how you installed it. But to keep up to date [PaperMod](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#installingupdating-papermod) I run this command:

```bash
git submodule update --remote --merge
```

### Social icons in footer

I added the social icons on every page (yes, because I want to be an influencer!) except on the homepage because it is already there by default for the **profile mode**. Ok yes I did it dorty.. instead of using the [partial templates](https://gohugo.io/templates/partials/) I modified the theme directly (will be improved in the future, I promised the 04.05.2024). Edit the file *theme/PaperMod/layouts/partials/footer.html* as follows.

{{< highlight html >}}
{{- if not (.Param "hideFooter") }}
<footer class="footer">
    {{ if not .IsHome }}
        {{ partial "social_icons.html" .Params.socialIcons }}
    {{ end }}
[...]
{{< / highlight >}}

### Privacy policy in footer

The [privacy policy](/privay) will be added to the footer in the same way as the [social icons](#social-icons-in-footer).

### CSS

To add your own custom CSS, create the following folder structure:

{{< highlight bash >}}
mkdir assets/css/extended
touch assets/css/extended/custom.css
{{< / highlight >}}

What I customised:

* The links change colour on hover.
* Social icons animate and change colour on hover
* Removed the ugly underline for social icons in the footer
* Removed haecoded blank space on the posts footer

{{< gist Schwitzd 1a2f2b3fe7707c5fb0e43bdcea607525 >}}

### 404 page

I choose to use the default PaperMod 404 page as the site is [hosted on Azure](/architecture) in the [staticwebapp.config.json](/staticwebapp.config.json) file I added:

{{< highlight json >}}
  "responseOverrides": {
    "404": {
      "rewrite": "/404"
    }
  }
{{< / highlight >}}
