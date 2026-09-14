# Joey Henkel's Blog

A minimalist, fast, and clean technical blog built with [Astro](https://astro.build), deployed to GitHub Pages.

## Features

- 🚀 **Fast & Minimal** — Built with Astro for blazing-fast performance
- 🌓 **Dark Mode** — Automatic dark mode support with system preference detection
- 📱 **Responsive** — Looks great on all devices
- 📝 **Markdown Posts** — Write content in simple Markdown
- 🔍 **Search Ready** — Configured for Pagefind search integration
- 📊 **Analytics** — Google Analytics integration ready
- 🔄 **Auto Deploy** — GitHub Actions automatically deploys on push to main

## Getting Started

### Local Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev
# Visit http://localhost:3000
```

### Building & Deployment

```bash
# Build for production
npm run build

# Preview production build locally
npm run preview
```

Push to `main` branch and GitHub Actions automatically deploys!

## Managing Content

### Writing Blog Posts

Create new `.md` files in `src/content/blog/`:

```markdown
---
title: "Your Post Title"
description: "Brief description for SEO"
pubDate: "Sep 14 2026"
---

Your content here in Markdown...
```

### Key Pages

- **About** — `src/pages/about.astro` — Professional background
- **Certifications** — `src/pages/certifications.astro` — Credentials
- **Home** — `src/pages/index.astro` — Landing page

### Site Config

Edit `src/consts.ts` for site title and description.

## Configuration

### Google Analytics

Update tracking ID in `src/components/BaseHead.astro` (replace `G-XXXXXXXXXX` with your ID).

### Styles

Global styles in `src/styles/global.css` use CSS variables that adapt to dark/light mode automatically.

## Project Structure

```
src/
├── components/     # Header, Footer, etc.
├── content/blog/   # Your blog posts
├── layouts/        # Page layouts
├── pages/          # Site pages
├── styles/         # Global styles
└── consts.ts       # Site config
```

## Publishing Workflow

1. Write post in `src/content/blog/`
2. Test locally: `npm run dev`
3. Commit: `git add . && git commit -m "Add new post"`
4. Push: `git push origin main`
5. GitHub Actions deploys automatically (1-2 minutes)
6. Visit https://joeyhenkel.github.io

## Useful Markdown Tips

```markdown
# Heading 1
## Heading 2

**Bold text** and *italic text*

- Bullet list
- Another item

1. Numbered list
2. Second item

[Link text](https://example.com)

> Quote or callout

\`\`\`javascript
// Code block
const x = 42;
\`\`\`
```

## Troubleshooting

- **Build fails?** Run `npm install` and check Node version (22+)
- **Changes not showing?** Make sure you pushed to `main` branch
- **Clear cache** with Cmd+Shift+R or Ctrl+Shift+R

## Learn More

- [Astro Docs](https://docs.astro.build)
- [Markdown Guide](https://www.markdownguide.org)
- [GitHub Pages Docs](https://docs.github.com/en/pages)
