# Portfolio site

Single-file static site — just `index.html`. No build step, no dependencies to install.

## Before you deploy

The file has 5 placeholder spots for your live URL (used for social-media link
previews and SEO, not for how the page looks in a browser). Search the file for
`REPLACE-WITH-YOUR-DOMAIN` and swap it for your real domain once you know it
(e.g. `https://sudhanshu-vipin.vercel.app` or your custom domain).

You can do this either:
- **Before pushing**, by editing `index.html` directly, or
- **After your first deploy**, once you know the URL Vercel/Netlify assigns you,
  then push the update.

Everything else — layout, content, the Projects section, images — is ready as-is.

## Deploy options

**Vercel (recommended, easiest)**
1. Push this folder to a new GitHub repo.
2. Go to vercel.com → New Project → import the repo.
3. No settings needed — it's a static `index.html`, Vercel will detect and deploy it automatically.

**GitHub Pages**
1. Push this folder to a GitHub repo.
2. Repo Settings → Pages → Source: deploy from branch → `main` → `/root`.
3. Your site will be live at `https://<username>.github.io/<repo-name>/`.

**Netlify**
1. Push to GitHub, then New site from Git in Netlify, pick the repo.
2. Build command: none. Publish directory: `/` (root).

## Notes

- Everything (fonts, images, script) is inlined or loaded via CDN — there's nothing else to upload.
- If you later add real screenshots for the Deep Ocean / IKYK Games project cards, keep them in the same repo and reference them with a relative path (e.g. `./images/deep-ocean.png`).
