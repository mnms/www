# How to write blog post

## Filename

* All blog posts must exist under `/_posts/`.
* Each file must have the name `year-month-day-title.md`.
  * `year` is 4 digits, `month` and `day` are 2 digits each.
  * Do not use special characters or spaces as `title`. (Please replace the blank with-.)
  * Each post would be built as the following path. `blog/year/month/title.html`
  * Example. `/_posts/2020-02-20-sample-post.md` => `lightningdb.io/blog/2002/02/sample-post.html`

## Front Matter

* [Front matter] (https://jekyllrb.com/docs/front-matter/) should be present at the top of each post(file) as `variable: value` pair.
* Available variables and descriptions are as follows.

| Variable | Description |
| -------- | ----------- |
| layout   | Set the layout to use. This value must be `blog`. |
| title    | Title of the article. This is also exposed in the blog menu > article list. |
| author   | Author. This is also exposed in the blog menu > article list. |
| category | Category. This is also exposed in the blog menu > article list. Visitors can view articles of a certain category. |
| excerpt  | This is a summary that will be displayed in the blog menu list. If omitted, some or before `<!-- more -->` paragraphs are displayed. |
| link     | This is used when ONLY external links WITHOUT content are displayed in the blog menu list. |
| published | Decide whether to publish or not. If omitted, it will be posted. If set to `false`, the post is not built and exposed. |

* The following is an example of a front matter which would not publish.

```
  ---
  layout: blog
  title: Sample blog article
  author: someone
  category: some-category
  excerpt: |
    Some excerpt text here - this excerpt is exposed on blog list
  published: false
  ---
```

* The following is an example of an external link without content.

```
  ---
  layout: blog
  title: Lightning DB YouTube Link
  author: someone
  category: other-category
  link: https://www.youtube.com/channel/UCj0aAcbjw0O-vVIcPMneNSw
  excerpt: |
    Test external links.
  ---
```

## Contents

* Typically you could write posts in Markdown, HTML is also supported.
  * For the Markdown syntax, see [here](https://guides.github.com/features/mastering-markdown/).
* Diagrams and formulas can also be expressed using [mermaid](http://mermaid-js.github.io/mermaid/) and [MathJax](https://www.mathjax.org/).
  * For syntax for diagram, refer to [here](http://mermaid-js.github.io/mermaid/#/flowchart).
  * For formula, wrap MathML or TeX in `$...$` or `$$...$$` delimiters (or \(...\) and \[...\])
    * Example. `x=\frac{-b \pm \sqrt{b^2 - 4ac}}{2a}` is expressed as `$$x=\frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$`

## Attachments (or images)

* Attachments or images must exist under `/assets/blog/` directory.
* Create a directory such as `year/month/` with the same as filename and place the files.
