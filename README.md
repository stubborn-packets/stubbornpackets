# stubbornpackets

**From the lab to the lake — documenting real homelab engineering.**

Practical documentation on **Network Engineering**, **Infrastructure as Code**, and **Machine Learning** integration from a North Carolina homelab.

This is my public working notebook: real configs, scripts that actually ran, network diagrams, lessons learned, and experiments integrating ML into network operations.

No fluff. Just practical knowledge.

## Overview

- Fully custom Hugo site (heavily customized on top of PaperMod)
- Data-driven homepage using `data/homelab.yaml` and `data/featured.yaml` (designed for future automation)
- Custom layouts and partials for a tailored experience
- Vendored theme for reproducible builds

## Site Structure

- `content/` — Markdown content (blog posts, about page, etc.)
- `layouts/` — Custom templates (including a fully custom homepage)
- `data/` — Structured data for the homepage (lab inventory, featured experiments)
- `static/` — Static assets (images, etc.)
- `assets/css/extended/` — Custom styles
- `hugo.yaml` — Site configuration
- `themes/PaperMod/` — Vendored theme

See [deploy.md](deploy.md) for the full deployment process and IaC roadmap.

## Local Development

```bash
# Start the dev server (with drafts)
hugo server -D

# Clean production build
hugo --cleanDestinationDir --minify
```

## Deployment

**Current path:** Cloudflare Pages (builds automatically from source on push to `main`).

- Build command: `hugo`
- Output directory: `public`
- `public/` is intentionally **not committed** to git (see `.gitignore`).

Full instructions, build settings, and the roadmap toward Terraform/Ansible-driven content creation and deployment are in [deploy.md](deploy.md).

**Future goals:**
- GitHub Actions for CI/CD (placeholder workflow exists in `.github/workflows/`)
- Full IaC: Use Terraform and/or Ansible to scaffold new blog posts, update data files, and orchestrate deployments.

## Contributing / Notes

This is primarily a personal notebook, but pull requests for fixes, improvements, or interesting homelab ideas are welcome.

When adding content:
- New blog posts go in `content/blog/`
- Update `data/` files for homepage visibility where appropriate
- The site is built to be automation-friendly

## License

Content is licensed under CC BY-NC-SA 4.0 (or similar — to be finalized). Code/configs are MIT unless otherwise noted.

---

Built with [Hugo](https://gohugo.io/) + a custom setup. Deployed on Cloudflare.