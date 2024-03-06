---
title: "Publish Vscode Foam With Hugo"
date: 2024-03-06T11:58:47+02:00
draft: false
---

# Introduction

[Foambubble](https://foambubble.github.io/) is a good instrument to structure your data in a graph-oriented way in a manner of roam research, obsidian etc. For me, foam is an ancestor for org-roam, but with vscode flavor. In the original foam documentation, you can easily find examples and instructions on how to publish your notes with _Jekyll_, _mkdocs_ and _Gatsby_ engines, but not _Hugo_. Here is my example of foam notes integration to _Hugo_ static website.

# Implementation

## Hugo + foam repository structure 

One of the cool foam features is collaborative maintenance of your knowledge storage with git. Let's assume, that I don't want to share my Hugo blog codebase with anyone, but I want to share some foam notes root. To support independent repository control git submodules could be used as a solution. 

`content` directory of Hugo website can look like this:

```
./content/
├── notes
│   └── shared - git submodule root for shared foam notes
│       ├── _index.md
│       ├── note-1.md
│       ├── note-2.md
│       └── some-other-note.md
└── posts
    ├── post1.md
    └── post2.md
```

## Foam configuration

Actually, foam requires no configuration, the only thing that is required for Hugo is a special heading format. Let's add this header to the default template to foam `.foam/templates/new-note.md`.

```md
---
type: basic-note
title: "$FOAM_TITLE"
draft: false
foam_template:
    filepath: $FOAM_SLUG.md
---
```

This is a minimal example, there also can be added a timestamp, tags, etc. 

I use custom `filepath`, because it will produce nice clean links to the website. This will introduce a problem: the foam janitor will not build reference definition links for wikilinks, hence we can just disable it: `"foam.edit.linkReferenceDefinitions": "off"`.

### _index.md

Naming of `_index.md` is important, it will tell Hugo, that there is a list of articles around. To render the index page as a normal page it is possible to change the layout to `singe` in tags.

```md
---
title: "Shared notes index page"
layout: "single"
---
```

## Hugo configuration

### Direct markdown references

Hugo should work out of the box, the only thing we need to add is to enable support of relative links. It is available for Hugo at the moment of the article for a stable version. For older versions, a custom Hook could be set up.

```toml
[markup]
[markup.goldmark]
[markup.goldmark.renderHooks]
[markup.goldmark.renderHooks.link]
enableDefault = true
```

### Wikilinks

Unfortunately, wikilinks is not supported by Hugo, but it could be implemented with custom partial. Quinn Casey has a good example in his [blog post](https://quinncasey.com/hugo-wikilink-support/) I slightly modified it to support slug namings.

Long story short, all we need to do is just add partial and use `{{- partial "content-with-wikilinks" . -}}` instead of `.Content` for `single.html` layout.

```html
{{ $wikiregexWithText := "\\[\\[([^\\]\\|\\r\\n]+?)\\|([^\\]\\|\\r\\n]+?)\\]\\]" }}
{{ $wikiregex := "\\[\\[([^\\]\\|\\r\\n]+?)\\]\\]" }}

{{ $page := .Page }}
{{ $pageContent := .Content }}

{{ range ($wikilinks := .Content | findRE $wikiregex) }}
	{{ $link := . | replaceRE $wikiregex "$1" }}
	{{ $wikilink :=  printf "\\[\\[%s\\]\\]" $link }}
    {{ $anchorized := $link | anchorize }}

	{{ with relref $page $anchorized }}
		{{ $link := printf "%s%s%s%s%s" "<a href=\"" . "\">" ($.Site.GetPage $anchorized).Title "</a>"  }}
		{{ $pageContent = $pageContent | replaceRE $wikilink $link }}
	{{ end }}
{{ end }}

{{ range ($pageContent | findRE $wikiregexWithText) }}
	{{ $link := . | replaceRE $wikiregexWithText "$1" }}
	{{ $text := . | replaceRE $wikiregexWithText "$2" }}
	{{ $wikilink :=  printf "\\[\\[%s\\|%s\\]\\]" $link $text }}	
    {{ $anchorized := $link | anchorize }}

	{{ with relref $page $link }}
		{{ $link := printf "%s%s%s%s%s" "<a href=\"" . "\">" $text "</a>" }}
		{{ $pageContent = $pageContent | replaceRE $wikilink $anchorized }}
	{{ end }}
{{ end }}

{{ $pageContent | safeHTML }}
```

### Backlinks

TDB;
