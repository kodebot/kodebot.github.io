# kodebot learning portal


## Run locally
`hugo server -D`

`-D` includes draft documents as well

## Publish
`hugo --cleanDestinationDir` generates the site into the `docs` folder and removes stale files from it.

Commit the generated `docs` together with the source changes; GitHub Pages serves `docs` from master.
Create a pr to merge into master to publish the changes.

The site needs Hugo 0.146 or later (`brew install hugo`) and is built with 0.162.1.
There is no theme: templates are in `layouts` and styles in `assets/css`.

## How to create new blog post

`hugo new blog\<year>\<blog-title-in-kebab-case>\index.md`

Example:
`hugo new blog\2021\what-is-cardano-and-ada\index.md`

Note: This will ensure each post has its own folder with necessary images, attachments, etc


## How to publish or update a book

Books are self-contained HTML files kept at `static/books/<slug>/index.html`.
The build serves each one at `/books/<slug>/` unchanged, apart from the site's favicon and a "Kodebot / Books" link
at the top of its sidebar (see `layouts/books/page.html`). The link goes inside the book's `<div class="masthead">`,
and the build warns about a book that has none.
The Books page lists them all from the `books` front matter in `content/books/_index.md`.
The home page lists only the entries marked `home: true`, beside an "All books" link to the Books page.

To update *Java for C# Developers*:

1. In the book repo (`dotnet2java`), run `python3 build.py`
2. Copy `dist/handbook.html` to `static/books/java-for-csharp-developers/index.html`
3. Run `hugo --cleanDestinationDir` to regenerate `docs`, then commit and push

To update *From Docker to Kubernetes*:

1. In the book repo (`quick-guide-to-docker-kubernetees`), render the Mermaid diagrams to `dist/svgs.json`
   (an array of SVG strings in book order; see `scripts/build-site.py` for how) and run `python3 scripts/build-site.py`
2. Copy `dist/index.html` to `static/books/from-docker-to-kubernetes/index.html`
3. Run `hugo --cleanDestinationDir` to regenerate `docs`, then commit and push

To add another book, put its HTML at `static/books/<slug>/index.html` with a cover image beside it,
and add an entry to `books` in `content/books/_index.md`.
Add `home: true` to the entry only if the book should also appear on the home page.


## Videos

The Videos page and the home page's "Latest videos" are read from the kodebot YouTube channel's RSS feeds when the site is built,
so rebuild and publish to pick up new uploads. Feeds are cached for 10 minutes; `hugo --cleanDestinationDir --ignoreCache` forces a refetch.

To show another playlist, add its id to `playlists` in `content/videos/_index.md`; the order there is the order on the page.
Each YouTube feed returns at most 15 videos: the latest uploads for the channel, and the first 15 in playlist order for a playlist,
so a longer playlist shows those plus a "See all on YouTube" link.
If a feed can't be fetched the build still succeeds with a warning, and that playlist is left out.


## How to write consistently
Use `h3` (###) onwards for subtitles 
Always create a folder per post and keep all local files in the folder

Shortcodes available in content:
* `{{% notice warning %}}...{{% /notice %}}` for a callout: `note`, `info`, `tip` or `warning`
* `{{% img "image.png" "caption" height width %}}` for an image in the post folder
* `{{% colorbtn href="https://..." icon="fab fa-youtube" %}}Watch video{{% /colorbtn %}}` for a link button
* `{{< youtube VIDEO_ID >}}` for an embedded video


## Troubleshooting
1. if you have problem with using variable and want to see what is in it use `{{ .VARIABLE | jsonify}}`. This will print the variable and its value as json on the page
