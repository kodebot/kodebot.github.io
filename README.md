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

Books are self-contained HTML files served as-is from `static/books/<slug>/index.html`.
The Books page and the home page list them from the `books` front matter in `content/books/_index.md`.

To update *Java for C# Developers*:

1. In the book repo (`dotnet2java`), run `python3 build.py`
2. Copy `dist/handbook.html` to `static/books/java-for-csharp-developers/index.html`
3. Run `hugo --cleanDestinationDir` to regenerate `docs`, then commit and push

To add another book, put its HTML at `static/books/<slug>/index.html` with a cover image beside it,
and add an entry to `books` in `content/books/_index.md`.


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
