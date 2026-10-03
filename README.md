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

```bash
hugo new talks/opinions-as-a-product.md      # a talk, given or upcoming
hugo new conferences/kubecon-na-2026.md      # a conference you went to
```

Titles are lowercase and conversational. Set `categories` on articles.

For talks and conferences, set `date` to the day of the event — future dates
are fine and show as "upcoming". Keep the `publishDate` the archetype fills in:
it's what lets a future-dated page build at all. The homepage shows the next
upcoming talk, the latest past talk, and the latest past conference; the full
lists live on the community page. A scheduled deploy rebuilds daily so things
roll from upcoming to past on their own.

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
