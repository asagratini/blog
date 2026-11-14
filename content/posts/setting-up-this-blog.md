+++
title = 'Setting up this blog: Hugo, GitHub Pages and Cloudflare'
date = 2026-12-06T12:00:00+01:00
draft = false
tags = ['hugo', 'github-pages', 'cloudflare', 'dns', 'blogging']
summary = 'Why I picked a static site over WordPress, how it is built and published, and the handful of things that went wrong along the way.'
ShowToc = true
+++

Every SRE eventually ends up with a folder full of notes that would have
saved them a weekend, had they found them online first. This blog is my
attempt to publish mine. A few posts in, it seemed fitting to tell how the
blog itself was built, including the parts that didn't work the first time.

## Why not WordPress

The obvious way to start a blog is a CMS like WordPress, and for the first
hour it probably is faster. But a CMS earns its keep when many people,
often non-technical, publish frequently, with a team keeping the platform
and its plugins patched. For a single author who posts every now and then, it
mostly adds work: a database, constant updates, and one of the most attacked
platforms on the internet, which needs the most attention exactly during
the weeks I'm not writing.

A static site generator flips that around. Posts are Markdown files in a Git
repository, a build turns them into plain HTML, and the result can be served
from anywhere. There's no database, nothing to patch on the server, and
nothing to attack beyond static files. Publishing is a `git push`.

I picked **[Hugo](https://gohugo.io/)**, a single binary, fast, and packaged
for both FreeBSD and macOS, with the **PaperMod** theme.

## The moving parts

- **The content:** a public GitHub repository, with posts under
  `content/posts/` and the theme as a Git submodule.
- **The build:** a GitHub Actions workflow, based on Hugo's own example,
  that builds the site on every push to `main` and publishes it.
- **The hosting:** GitHub Pages, served through Fastly's CDN, free for
  public repositories.
- **The domain:** `blog.sagratini.net`, a CNAME to `asagratini.github.io`
  in Cloudflare, with GitHub issuing and renewing the HTTPS certificate.

One small detail that matters: the Hugo version in the workflow is pinned to
the one I use locally (0.168.0 at the time of writing), so the preview on my
machine and the published site are built identically.

## Writing a post

Each post is a Markdown file with a short header:

```toml
+++
title = 'When CARP, HAST and Spanning Tree meet at boot'
date = 2026-11-08T12:00:00+01:00
draft = true
tags = ['freebsd', 'carp', 'hast']
+++
```

While `draft = true`, the post only exists in my local preview,
`hugo server -D`, since production builds leave drafts out. When it reads
right, I flip it to `false`, commit and push, and a couple of minutes later
it's live.

## What went wrong (so you don't have to)

**The home page was an RSS feed.** My first deployment served Hugo's XML
feed instead of a web page, with an empty title. The build log explained
it: the PaperMod theme wasn't in the repository. I had added it locally, but
never committed it as a submodule, so the workflow's "initialize submodules"
step found nothing, and Hugo, without a theme, only generated the feed. The
fix:

```sh
git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
git add .gitmodules themes/PaperMod
git commit -m "Add the PaperMod theme as a submodule"
```

The same log also flagged that `languageCode` is deprecated since Hugo
0.158, replaced by `locale`.

**GitHub couldn't see my DNS record.** I set the custom domain in the
repository's Pages settings *before* creating the CNAME in Cloudflare, so
GitHub's first lookup found nothing, and resolvers cached that negative
answer for a while. Checking public resolvers directly
(`dig +short blog.sagratini.net CNAME @1.1.1.1`) confirmed the record was
fine, and GitHub picked it up shortly after.

**My own firewall blocked everyone.** My Cloudflare zone has a custom rule
that blocks everything except my network and GitHub's webhook ranges,
for services that only I should reach. A small program of mine rebuilds that
rule whenever my IP address changes, so editing it by hand would have been
undone at the next run. The fix went into the program itself, exempting the
blog's hostname from the expression:

```
http.host ne "blog.sagratini.net" and not (ip.src in {...})
```

Cloudflare's analytics made the problem obvious: the most requested paths
were the files of Cloudflare's own block page.

## Cloudflare in front, with care

I keep the DNS record **proxied**, so the site sits behind Cloudflare, with
three things worth knowing if you do the same:

- **SSL mode must be Full (strict).** With *Flexible*, Cloudflare talks to
  GitHub over plain HTTP, GitHub redirects to HTTPS, and visitors get an
  endless redirect loop.
- **Certificate renewals.** GitHub renews its Let's Encrypt certificate
  through an HTTP challenge on your domain. If a renewal ever fails, turning
  the proxy off temporarily lets it through.
- **It's not a wall.** GitHub Pages still serves the site directly to anyone
  who asks its servers for the hostname, so for a public blog, Cloudflare is
  a convenience, not a security boundary.

There's also a very Spanish caveat: during football matches, Spanish ISPs
block Cloudflare IP ranges used by pirate streams, taking plenty of
unrelated sites down with them. Serving the record DNS-only, straight from
GitHub's CDN, is the way around it, if that ever becomes a problem.

## Being found

Two surprises here. First, the site had visitors from four countries within
an hour of going live, before I'd told anyone. The reason: when GitHub
obtained the certificate, it was logged in the public **Certificate
Transparency** logs, which crawlers and security scanners watch for new
hostnames.

Second, getting into search results takes a deliberate step. Hugo generates
a `sitemap.xml`, and submitting it in **Google Search Console** (verified
with a DNS TXT record) had Google discover all the pages on the first read.
From then on, Google re-reads the sitemap on its own, so new posts get picked
up without me doing anything.

## No cookies, no banner

I wanted to know roughly how many people read the blog, without a cookie
consent banner. Google Analytics sets cookies, and since I'm based in the
EU, using it would mean asking every visitor for consent first. Cookieless
analytics, like Cloudflare Web Analytics, avoid the question entirely, and
show what I actually care about: page views, and where readers come from.

## Takeaways

- **Match the tool to the job.** A one-person technical blog doesn't need a
  CMS; Markdown, Git and a static generator are less work, and safer.
- **Read the build log first.** Every problem here was explained in plain
  words by the tool that hit it.
- **Check every layer.** DNS, CDN, firewall rules and certificates each had
  their own opinion about whether the site should be reachable.
- **Pin your versions.** The same Hugo locally and in CI means no surprises
  between preview and production.
