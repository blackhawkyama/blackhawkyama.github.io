# BlackhawkYama — personal site & blog

A Jekyll site that deploys automatically to GitHub Pages.

## Go live

The repo on GitHub must be named `blackhawkyama.github.io`. Then:

```bash
git push -u origin main
```

(git is already initialized, committed, and the remote is set.)

Live at `https://blackhawkyama.github.io` within a minute or two.
Confirm under **Repo → Settings → Pages** (Source = "Deploy from a branch",
branch `main`, folder `/root`).

## Preview locally (optional)

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000
```

## Add a post

Drop a file in `_posts/` named `YYYY-MM-DD-title.md` with front matter. Push.

## Upgrade to a sharper theme later

`minima` (current) is the bulletproof default. For a polished technical-blog
look, swap to one of these in `_config.yml`:

- **Minimal Mistakes** — `remote_theme: "mmistakes/minimal-mistakes"`
- **Chirpy** — best to start from the
  [chirpy-starter](https://github.com/cotes2020/chirpy-starter) template
  rather than retrofitting; it ships the assets Chirpy needs.

## Custom domain (optional, ~$10/yr)

1. Buy `blackhawkyama.com`.
2. Add a file named `CNAME` containing just: `blackhawkyama.com`
3. Point DNS at GitHub Pages and enable HTTPS in Settings → Pages.
