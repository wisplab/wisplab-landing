# wisplab.org

Source for the [wisplab.org](https://wisplab.org) landing page.

## Stack

- Static HTML, no build step
- Deployed via Cloudflare Pages
- Custom domain: wisplab.org

## Local preview

```bash
python3 -m http.server 8000
# Open http://localhost:8000
```

## Deploy

Pushes to `main` auto-deploy via Cloudflare Pages.

## Files

- `index.html` — landing page
- `logo.png` — favicon + hero logo (drop file here, 256×256+ recommended)
