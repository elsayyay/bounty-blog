# Bug Bounty Blog

Personal security research blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Deployed to GitHub Pages via GitHub Actions.

## Quick Start

### 1. Install Hugo

```bash
# macOS
brew install hugo

# Ubuntu/Debian
sudo snap install hugo

# Windows (via Chocolatey)
choco install hugo-extended
```

### 2. Clone & Add Theme

```bash
git clone https://github.com/elsayyay/bounty-blog.git
cd bounty-blog

# Add PaperMod as a git submodule
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 3. Configure

Edit `hugo.toml` — replace all `YOUR_*` placeholders with your actual info:
- `baseURL` → your GitHub Pages URL
- Social links → your GitHub, HackerOne, LinkedIn, email

Edit `content/about.md` — update the same placeholders.

### 4. Run Locally

```bash
hugo server -D
# Open http://localhost:1313
```

### 5. Write a New Post

```bash
hugo new posts/my-new-writeup.md
```

This creates a new file from the writeup template in `archetypes/posts.md`. Fill it in, then set `draft: false` when ready to publish.

### 6. Deploy to GitHub Pages

1. Push this repo to GitHub
2. Go to **Settings → Pages → Source** → set to **GitHub Actions**
3. Push to `main` — the workflow in `.github/workflows/hugo.yml` builds and deploys automatically
4. Your site will be live at `https://elsayyay.github.io/bounty-blog/`

## Project Structure

```
bounty-blog/
├── archetypes/
│   └── posts.md              # Writeup template (used by `hugo new`)
├── content/
│   ├── about.md               # About page
│   ├── search.md              # Search page
│   └── posts/                 # All writeups go here
│       └── hello-world.md     # Starter post
├── static/
│   └── images/                # Screenshots, diagrams for posts
├── .github/
│   └── workflows/
│       └── hugo.yml           # GitHub Actions deploy workflow
├── hugo.toml                  # Site configuration
└── README.md
```

## Writing Tips

- Put screenshots in `static/images/` and reference as `/images/filename.png`
- Use tags consistently: `xss`, `stored-xss`, `oauth`, `idor`, `auth-bypass`, `js-reversing`, etc.
- Always set `draft: true` until disclosure is approved
- The writeup archetype has the full template — just fill in the sections

## Custom Domain (Optional)

1. Buy a domain (Cloudflare Registrar is cheapest)
2. Add a `CNAME` file to `static/` containing your domain
3. Update `baseURL` in `hugo.toml`
4. Configure DNS: CNAME record pointing to `YOUR_USERNAME.github.io`
