# kodebot learning portal


## Run locally
`hugo run -D`

`-D` includes draft documents as well

run hugo to generate the content -> it is generated into docs folder

create pr to merge the generated content into master branch to publish the changes

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
3. Run `GOMAXPROCS=1 hugo` to regenerate `docs`, then commit and push

To add another book, put its HTML at `static/books/<slug>/index.html` with a cover image beside it,
and add an entry to `books` in `content/books/_index.md`.

The site is built with Hugo 0.80.0. Newer Hugo versions do not work with the current theme.
Hugo 0.80.0 intermittently panics while rendering markdown; `GOMAXPROCS=1` makes the build reliable.


## How to write consistently
Use `h3` (###) onwards for subtitles 
Always create a folder per post and keep all local files in the folder


## Troubleshooting
1. if you have problem with using variable and want to see what is in it use `{{ .VARIABLE | jsonify}}`. This will print the variable and its value as json on the page