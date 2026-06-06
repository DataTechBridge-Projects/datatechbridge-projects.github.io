# Daily Tech Insights Blog

A Jekyll-based blogging platform for daily insights on prompt engineering, security, and AI topics.

## Setup Instructions

### Prerequisites
- Ruby 2.7 or higher
- Git
- GitHub account

### Local Setup

1. **Install Ruby dependencies**:
```bash
bundle install
```

2. **Serve locally**:
```bash
bundle exec jekyll serve
```

Visit `http://localhost:4000` in your browser.

3. **Build for production**:
```bash
bundle exec jekyll build
```

## How to Add Daily Posts

### Creating a New Post

1. Create a new file in `_posts/` with the naming convention: `YYYY-MM-DD-post-title.md`
2. Add front matter (metadata) at the top
3. Write your content

### Post Template

```markdown
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
categories: [topic1, topic2]
---

Your content goes here in Markdown format.
```

### Example
- Date: 2026-06-05 → Filename: `2026-06-05-my-topic.md`
- Date: 2026-06-06 → Filename: `2026-06-06-another-topic.md`

## Deploying to GitHub Pages

### Option 1: GitHub Pages (Recommended)

1. **Create a GitHub repository**:
   - If deploying to `username.github.io`: Name it `username.github.io`
   - Otherwise: Name it anything and enable GitHub Pages in Settings → Pages

2. **Push your code**:
```bash
git remote add origin https://github.com/yourusername/your-repo.git
git branch -M main
git push -u origin main
```

3. **Enable GitHub Pages** (if not `username.github.io`):
   - Go to Settings → Pages
   - Select "Deploy from a branch"
   - Choose `main` branch

4. **GitHub will automatically build and deploy your site**
   - Your blog will be live at:
     - `https://username.github.io` (if repo is named `username.github.io`)
     - `https://username.github.io/repo-name` (otherwise, update `baseurl` in `_config.yml`)

### Update _config.yml

Replace these values in `_config.yml`:
```yaml
title: Your Blog Title
description: Your blog description
url: "https://yourusername.github.io"
github_username: yourusername
twitter_username: yourhandle
```

## Daily Blogging Workflow

### Quick Daily Process

```bash
# 1. Create new post file
# 2. Edit _posts/YYYY-MM-DD-topic.md
# 3. Save and commit
git add _posts/
git commit -m "Add new post: Your Topic"
git push origin main
```

### Batch Writing Posts

Write posts ahead of time:
```bash
_posts/
├── 2026-06-05-prompt-engineering.md
├── 2026-06-06-prompt-injection.md
├── 2026-06-07-topic-3.md
└── 2026-06-08-topic-4.md
```

## Current Posts

1. **Prompt Engineering** (2026-06-05) - Principles, techniques, and applications
2. **Prompt Injection** (2026-06-06) - Security threats and defenses

## Customization

### Change Theme
Edit `_config.yml` to use a different Jekyll theme:
```yaml
theme: minima  # or another theme
```

Popular themes: `minima`, `jekyll-theme-cayman`, `jekyll-theme-slate`

### Add Custom Styling
Create `assets/main.scss` to override theme styles.

### Add Navigation Menu
Create `_data/navigation.yml`:
```yaml
- name: Home
  link: /
- name: About
  link: /about
```

## Markdown Tips

```markdown
# Heading 1
## Heading 2
### Heading 3

**Bold text**
*Italic text*

- Bullet point
- Another point

1. Numbered list
2. Second item

[Link text](https://example.com)

![Image alt](image.jpg)

> Blockquote

`inline code`

\`\`\`python
# Code block
print("Hello")
\`\`\`
```

## Troubleshooting

### Build issues
```bash
bundle update
bundle install
bundle exec jekyll serve
```

### Post not appearing
- Check filename format: `YYYY-MM-DD-title.md`
- Ensure front matter is valid YAML
- Check date isn't in the future (Jekyll hides future posts by default)

### GitHub Pages not updating
- Push to the correct branch (usually `main`)
- Check repository Settings → Pages for build status
- Look at GitHub Actions for error messages

## Resources

- [Jekyll Documentation](https://jekyllrb.com/)
- [GitHub Pages Docs](https://docs.github.com/en/pages)
- [Markdown Guide](https://www.markdownguide.org/)
- [YAML Front Matter](https://jekyllrb.com/docs/front-matter/)

## License

MIT License - Feel free to modify and use this template.
