# Nikhil Adhikari — Portfolio Site

Personal portfolio and blog, built with [Hugo](https://gohugo.io/) and the
[Stack](https://github.com/CaiJimmy/hugo-theme-stack) theme. Deploys to GitHub
Pages automatically.

## Requirements

- **Hugo Extended ≥ 0.157** (this site was built with 0.160). The plain (non-extended)
  build will NOT work — the theme needs SCSS compilation.

## Preview locally

```bash
hugo server -D
```

Then open http://localhost:1313. Pages update live as you edit.

## Add a blog post

```bash
hugo new post/my-new-post/index.md
```

Edit the new file, set `draft: false` (or remove the draft line), add
`categories` and `tags`, and drop images in the same folder.

## Deploy

Every push to the `main` branch triggers the GitHub Actions workflow in
`.github/workflows/hugo.yml`, which builds the site and publishes it to GitHub
Pages. No manual build needed.

## Structure

```
config/_default/   Site config (config.toml) and menus (menu.toml)
content/post/      Blog posts
content/page/      Standalone pages: about, projects, archives, search
assets/img/        avatar.jpg — replace with your own photo
assets/icons/      Custom SVG icons (linkedin, code) added on top of the theme
themes/stack/      The theme (committed directly — no submodule to manage)
```

## Things to personalize

- Replace `assets/img/avatar.jpg` with a real photo (square, ~600×600).
- Edit the sidebar `subtitle` and `description` in `config/_default/config.toml`.
- Update social links in `config/_default/menu.toml`.
