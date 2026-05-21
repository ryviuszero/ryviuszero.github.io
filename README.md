# ryviuszero.github.io

Personal Jekyll blog for `https://ryviuszero.github.io`.

## Local development

Use Ruby 3.2, matching the GitHub Pages deployment workflow.

```powershell
bundle install
bundle exec jekyll serve
```

Open `http://127.0.0.1:4000`.

To run a one-off build:

```powershell
bundle exec jekyll build
```

## Content

- Chinese posts live in `_posts` with `lang: zh`.
- English posts live in `_posts` with `lang: en`.
- Pages under `en/` provide the English index and about page.
