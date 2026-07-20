# blog.rotella.cloud

Personal/company blog built with [Jekyll][jekyll] and the [Chirpy][chirpy] theme, deployed to GitHub Pages via `.github/workflows/pages-deploy.yml`. Every push to `main` (except changes limited to `.gitignore`, `README.md`, or `LICENSE`) triggers a build and deploy — there is no manual publish step.

## How to add a new post

1. **Add any images first**, into `assets/img/`. Use descriptive filenames (e.g. `homelab-header-03.png`).
2. **Create the post file** in `_posts/`, named:

   ```
   YYYY-MM-DD-title-NN.md
   ```

   - `YYYY-MM-DD` is the publish date.
   - `title` is a short slug for the post/series (e.g. `homelab`).
   - `NN` is a zero-padded sequence number when the post is part of a series (e.g. `homelab-01`, `homelab-02`). Drop the `-NN` suffix for standalone posts.

   Example: `_posts/2025-02-19-homelab-02.md`.

3. **Fill in the front matter** (see `_posts/.boilerplate` for a blank template):

   ```yaml
   ---
   title: Post Title
   date: YYYY-MM-DD HH:MM:SS +0100
   categories: [Posts]
   tags: [tag-one, tag-two]     # lowercase only
   ---
   ```

4. **Reference images** from the post body using the `assets/img/` path:

   ```markdown
   ![Desktop View](assets/img/homelab-header-03.png)
   ```

5. **Preview locally** (optional but recommended before pushing):

   ```shell
   bundle install
   bundle exec jekyll serve
   ```

   Then open http://127.0.0.1:4000.

6. **Commit and push to `main`.** The `Build and Deploy` GitHub Actions workflow builds the site with Jekyll, runs `htmlproofer` link/HTML checks, and deploys to GitHub Pages automatically. No manual release step is needed.

## Repo layout

```
.
├── _posts/          # blog posts (YYYY-MM-DD-title-NN.md)
├── assets/img/      # post images
├── _tabs/           # static pages (About, Archives, etc.)
├── _data/, _includes/, _layouts/, _plugins/, _config.yml   # Chirpy theme overrides
└── .github/workflows/pages-deploy.yml   # build + deploy pipeline
```

## Theme docs

This repo is based on the [Chirpy Starter][chirpy-starter]. For theme-level configuration options (not the day-to-day posting workflow above), see the [Chirpy wiki][chirpy-wiki].

## License

This work is published under the [MIT][mit] License.

[jekyll]: https://jekyllrb.com/
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[chirpy-starter]: https://github.com/cotes2020/chirpy-starter
[chirpy-wiki]: https://github.com/cotes2020/jekyll-theme-chirpy/wiki
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
