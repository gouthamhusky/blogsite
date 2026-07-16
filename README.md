# blog

A small, text-first Hugo blog. System fonts, hand-written CSS, no JavaScript.

## running locally

Install Hugo (extended), then:

```bash
hugo server -D      # -D includes drafts; drop it for a clean preview
```

Open http://localhost:1313.

## writing

```bash
hugo new blog/some-notes-on-x.md   # a weekly article
hugo new til/til-thing-i-learned.md  # a daily snippet
```

Titles are lowercase and conversational. Set `categories` on articles — they
drive the grouping on the homepage.

### shortcodes

A callout:

```
{{</* note "gotcha" */>}}
This renders as a bordered box with a `# gotcha` label.
{{</* /note */>}}
```

A code block with a filename bar:

```
{{</* file "deploy.sh" */>}}
` ` `bash
echo hi
` ` `
{{</* /file */>}}
```

## deploying

Push to `main`. The workflow in `.github/workflows/deploy.yml` builds the site
and publishes it to GitHub Pages.

**One-time setup:** repo → Settings → Pages → Build and deployment → Source →
**GitHub Actions**. Also set `baseURL` in `hugo.toml` to your published URL.
