# Traced — Website

Static landing page for [traced.ai](https://traced.ai). Deployed via GitHub Pages.

## Deploy on GitHub Pages

1. Go to your repo **Settings → Pages**
2. Under **Source**, select **Deploy from a branch**
3. Set the branch to `main` and folder to `/docs/website`
4. Click **Save** — the site will be live at `https://<your-username>.github.io/traced.ai/`

### Custom domain (optional)

To use `traced.ai` as the domain:

1. Add a `CNAME` file in this folder containing `traced.ai`
2. In your DNS provider, add a CNAME record pointing `traced.ai` to `<your-username>.github.io`
3. In GitHub Pages settings, enter `traced.ai` as the custom domain and enable HTTPS

## Local preview

Open `index.html` directly in a browser — no build step needed.
