# blog

Hawkie's web-log. Main purpose is personal memo.

## 🚀 Features

- Built with [Quarto](https://quarto.org/)
- Automated publishing with GitHub Actions
- Hosted on GitHub Pages

## 📝 Adding New Posts

### 1. Create a new branch

```bash
git checkout -b update/your-article-name
```

### 2. Write your article

Create a new Markdown file in `raws/`:

```bash
cat > raws/your-article.md << 'EOF'
---
title: "Your Article Title"
author: "Your Name"
date: "2026-01-03"
categories: [category1, category2]
---

# Introduction

Your content here...
EOF
```

### 3. Commit and push

```bash
git add raws/your-article.md
git commit -m "Add: Your article title"
git push origin update/your-article-name
```

### 4. Create Pull Request

- Validation CI will automatically run
- If it passes, create a PR to `main`
- Merge the PR to publish

## 🏗️ Local Development

### Prerequisites

- [Quarto](https://quarto.org/docs/get-started/) installed

### Render locally

```bash
quarto render raws/ --to html
```

Open `_site/index.html` in your browser.

### Preview server

```bash
quarto preview raws/
```

## 📁 Directory Structure

```
blog/
├── raws/              # Markdown source files
├── _site/             # Generated HTML (gitignored)
├── _quarto.yml        # Quarto configuration
└── .github/workflows/ # CI/CD workflows
```

## 🔒 Security

This is a public repository. **Do not commit**:

- API keys or tokens
- Passwords or secrets
- Personal information
- Confidential data

## 📚 Documentation

- [SPEC.md](SPEC.md) - System specification
- [PLAN.md](PLAN.md) - Implementation plan

## 📄 License

See [LICENSE](LICENSE)
