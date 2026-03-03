# GitHub Pages Quick Start Guide

## What is docs-gh-pages?

The `docs-gh-pages` folder contains complete deployable static website files, designed for GitHub Pages.

## Directory Structure

```
docs-gh-pages/
├── .github/
│   └── workflows/
│       └── deploy.yml       # GitHub Actions auto-deploy workflow
├── .nojekyll                # Disable Jekyll processing
├── index.html               # Main page (docsify entry)
├── README.md                # Homepage content
├── _sidebar.md              # Sidebar navigation
├── coverpage.md             # Cover page (optional)
├── DEPLOY.md                # Detailed deployment guide
└── docs/                    # Chapter documents
    ├── 01_basics.md
    ├── 02_react.md
    ├── 03_tools.md
    ├── 04_memory.md
    ├── 05_mcp.md
    └── 06_skills.md
```

## Quick Deploy (3 steps)

### Method 1: Using a standalone repository

```bash
# 1. Enter docs-gh-pages directory
cd docs-gh-pages

# 2. Initialize git and push
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main

# 3. Enable GitHub Pages in GitHub repo Settings → Pages
#    Select: Deploy from a branch → main branch → / (root)
```

### Method 2: Using GitHub Actions auto-deploy

If you want to keep the full project structure:

```bash
# 1. Push entire project to GitHub
git add .
git commit -m "Add docs-gh-pages"
git push origin main

# 2. GitHub Actions will auto-detect docs-gh-pages and deploy
#    (Requires workflow trigger configuration)
```

## Access Your Website

After deployment (about 1-2 minutes), visit:

```
https://YOUR_USERNAME.github.io/YOUR_REPO_NAME/
```

## Verification Checklist

After deployment, check:

- [ ] Homepage displays correctly
- [ ] Sidebar navigation is visible
- [ ] Chapter links work and jump correctly
- [ ] Code highlighting displays properly
- [ ] Mermaid charts render correctly
- [ ] Search function works

## Custom Configuration

### Modify Website Title

Edit `index.html`:
```html
<title>Your Website Title</title>
<meta name="description" content="Your description">
```

### Modify Navigation Menu

Edit `_sidebar.md`:
```markdown
- [Home](/README.md)
- [Chapter 1](/docs/01_basics.md)
```

### Add Custom Domain

1. Go to GitHub repo Settings → Pages → Custom domain
2. Enter your domain
3. Add CNAME record at your DNS provider

## Update Documents

```bash
# Modify markdown files in docs/
# ... edit files ...

# Commit and push
git add .
git commit -m "Update: describe changes"
git push origin main

# GitHub Pages will auto-rebuild (1-2 minutes)
```

## Troubleshooting

### 404 Error
- Wait 2-3 minutes
- Confirm GitHub Pages is enabled
- Check if index.html exists

### Missing Styles
- Clear browser cache
- Check if CDN links are accessible

### Link 404
- Confirm links use `.md` extension
- Check if paths are correct (starting with `/`)

## Technical Notes

- **Route Mode**: hash mode (starts with `#`), best for GitHub Pages
- **CDN**: Uses jsDelivr for docsify resources
- **Build**: No build required, pure static files
- **.nojekyll**: Tells GitHub to use existing HTML directly

## Resources

- [GitHub Pages Docs](https://docs.github.com/en/pages)
- [docsify Documentation](https://docsify.js.org/)
- [Detailed Deployment Guide](./DEPLOY.md)
