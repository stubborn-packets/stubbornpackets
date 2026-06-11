# Deployment

This document describes how to build and deploy the **stubbornpackets** Hugo site.

The site is a fully custom, data-driven Hugo site focused on Infrastructure as Code, network engineering, and ML in the homelab. The long-term goal is to treat the entire site (including new blog post creation and updates) as infrastructure-as-code using Terraform and/or Ansible.

## Current Architecture

- **Source of truth**: Everything under `content/`, `layouts/`, `data/`, `static/`, `assets/`, `hugo.yaml`, and the vendored theme in `themes/PaperMod/`.
- **Build output**: The `public/` directory (intentionally **not** committed to git — see `.gitignore`).
- **Hosting**: Cloudflare Pages (or Cloudflare Workers in the future).
- **Build process**: Hugo generates a fully static site.

**Do not commit `public/`**. It is a build artifact and will be regenerated on every deployment.

## Local Development

### Prerequisites

- [Hugo Extended](https://gohugo.io/getting-started/installing/) (the project uses features that benefit from the extended version)
- Git

### Run the development server

```bash
hugo server -D
```

- Visit http://localhost:1313
- Live reload is enabled by default
- Use `-D` (or `--buildDrafts`) to see draft content

### Clean build locally

```bash
hugo --cleanDestinationDir --minify
```

The output will be in the `public/` directory.

## Deploying to Cloudflare Pages (Current Recommended Path)

Cloudflare Pages has native Hugo support and is the simplest way to deploy while we build toward full IaC automation.

### 1. Initial Setup

1. Push your changes to GitHub (or the Git host connected to Cloudflare).
2. Go to [Cloudflare Pages](https://pages.cloudflare.com/) → **Create a project** → **Connect to Git**.
3. Select your repository (`stubbornpackets`).
4. In the build settings:
   - **Framework preset**: Hugo
   - **Build command**: `hugo`
   - **Build output directory**: `public`
5. (Optional but recommended) Set a specific Hugo version under **Environment variables**:
   - `HUGO_VERSION`: `0.163.0` (or the latest extended version you are using locally)

6. Click **Save and Deploy**.

Cloudflare will automatically build and deploy on every push to the production branch (usually `main`).

### 2. Custom Domain

- After the first successful deployment, go to **Custom domains** in your Pages project.
- Add `stubbornpackets.com` (and `www.stubbornpackets.com` if desired).
- Follow Cloudflare's DNS instructions (usually just a CNAME).

The `baseURL` in `hugo.yaml` is already set to `https://stubbornpackets.com/`.

### 3. Preview Deployments

Cloudflare automatically creates preview deployments for pull requests. This is very useful for reviewing changes before merging.

## Adding New Content (Blog Posts, Updates)

1. Create new content in `content/blog/` (use the archetype if desired: `hugo new content/blog/my-new-post.md`).
2. Update supporting data files if needed (`data/featured.yaml`, `data/homelab.yaml`, etc.).
3. Commit and push the **source files only**.
4. Cloudflare Pages (or your future automation) will:
   - Run `hugo`
   - Deploy the new `public/` output

This workflow is already friendly for future automation — you (or a script) only need to modify source files.

## Future: Full Infrastructure as Code

The project is being built with eventual full IaC in mind:

- **Content creation** — Use Terraform or Ansible to scaffold new blog posts from templates.
- **Data updates** — Update `data/homelab.yaml` or `data/featured.yaml` programmatically.
- **Deployment orchestration** — Trigger Cloudflare deploys via Terraform (Cloudflare provider) or Ansible.
- **Site infrastructure** — Manage the Cloudflare Pages project, custom domains, DNS records, cache settings, etc. via Terraform.

Example future commands / flows (to be implemented):

- A Terraform module that creates a new post file and opens a PR.
- An Ansible playbook that updates lab inventory data and triggers a deploy.
- GitHub Actions (or direct Cloudflare integration) that runs on specific events.

See the roadmap in the main project notes for more details (to be expanded).

## Hugo Version & Theme Notes

- The site uses a **vendored copy** of the PaperMod theme (`themes/PaperMod/`).
- This means builds do **not** require `hugo mod` or external theme downloads.
- Use the **Extended** version of Hugo for best compatibility with image processing and custom CSS.

## Troubleshooting

- **Site looks outdated after push**: Check the Cloudflare Pages build logs. Make sure the build command succeeded and `public/` was populated.
- **Broken styles or missing assets**: Confirm you ran a clean build (`hugo --cleanDestinationDir`). Hashed CSS files are regenerated on every build.
- **Drafts not showing in production**: Remove `draft: true` from frontmatter or ensure the build command does not use `--buildDrafts`.
- **Custom domain not working**: Verify DNS records and that the domain is added in the Cloudflare Pages project (not just the domain registrar).
- **Local vs production differences**: Cloudflare runs a clean build in their environment. Always test with `hugo --cleanDestinationDir --minify` locally before pushing.

## Quick Reference Commands

```bash
# Development
hugo server -D

# Production build (local testing)
hugo --cleanDestinationDir --minify

# Check what will be deployed
ls -la public/
```

---

This document will be updated as we move toward full Terraform/Ansible-driven content creation and deployment. GitHub Actions workflows will be added in a future iteration.