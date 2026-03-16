# K8s Mastery Guide — Self-Hosted

A single-file, browser-based Kubernetes learning guide with sidebar navigation, dark/light mode, search, code copy, progress tracking, and section completion markers.

## Hosting on GitHub Pages (Free — 2 minutes)

### Step 1: Create a GitHub repository

```bash
# Option A: From the command line
gh repo create k8s-mastery-guide --public
cd k8s-mastery-guide

# Option B: Via github.com
# Go to https://github.com/new
# Name it "k8s-mastery-guide", make it public, click "Create repository"
# Then clone it:
git clone https://github.com/YOUR_USERNAME/k8s-mastery-guide.git
cd k8s-mastery-guide
```

### Step 2: Add the files

```bash
# Copy index.html into the repo root
cp /path/to/index.html .

# Commit and push
git add .
git commit -m "Initial K8s mastery guide"
git push origin main
```

### Step 3: Enable GitHub Pages

1. Go to your repo on GitHub: `https://github.com/YOUR_USERNAME/k8s-mastery-guide`
2. Click **Settings** (top tab bar)
3. Click **Pages** (left sidebar)
4. Under "Source", select **Deploy from a branch**
5. Set branch to `main` and folder to `/ (root)`
6. Click **Save**

Your site will be live in ~60 seconds at:
```
https://YOUR_USERNAME.github.io/k8s-mastery-guide/
```

That's it! Bookmark it and you have your own K8s reference site.

---

## Alternative: Run Locally (No GitHub)

If you just want to open it in your browser without any hosting:

```bash
# Option 1: Just open the file directly
# Double-click index.html in your file manager
# or:
open index.html          # macOS
xdg-open index.html      # Linux
start index.html         # Windows

# Option 2: Local dev server (if you want live reload)
# Python
python3 -m http.server 8000
# Then open http://localhost:8000

# Node.js
npx serve .
# Then open http://localhost:3000
```

> **Note:** The `localStorage`-based progress tracking (marking sections complete, theme preference) works fine when opened as a local file or served over HTTP. It persists across browser sessions.

---

## Features

- **Sidebar navigation** — jump to any section instantly
- **Search** — filter sections by keyword
- **Dark / Light mode** — toggle with the sun/moon button, persists across sessions
- **Progress bar** — shows reading progress as you scroll
- **Section completion tracking** — mark sections done, state saved in localStorage
- **Code copy buttons** — one-click copy on all code blocks
- **Mobile responsive** — hamburger menu on small screens
- **Zero dependencies at runtime** — just one HTML file + CDN for marked.js and fonts
- **Self-contained** — all markdown content is embedded in the HTML

## Updating the Guide

The markdown is embedded inside the `<script id="mdSource" type="text/markdown">` tag in `index.html`. To update the content:

1. Edit the markdown between the `<script>` tags
2. Commit and push — GitHub Pages auto-deploys

Or, if you prefer to keep the markdown separate:
1. Edit `k8s-mastery-guide.md`
2. Re-embed it (replace the content inside the script tag)
3. Push

## License

Personal use. Customize freely.
