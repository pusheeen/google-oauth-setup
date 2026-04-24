# google-oauth-setup

End-to-end Google OAuth 2.0 setup for a web app. Automates every part
that **can** be automated, with a tight runbook for the two Google Cloud
Console steps that require a human click.

## Why this exists

Setting up Google OAuth from scratch ate ~2 hours of my time on
2026-04-23. Every minute was wasted on one of four gotchas that are not
in any Google doc, because Google's own tooling doesn't acknowledge they
exist.

| # | Gotcha | What happens if you don't know it |
|---|---|---|
| A | Chrome cookies need the `.google.com` dot-prefix | Cookie import returns 0 cookies, you waste 10 minutes debugging the import tool |
| B | Client Secret is only visible in the creation modal | You try `gcloud` for 30 minutes, discover the API is dead (see D), give up |
| C | `echo "$VAL" \| vercel env add` adds `\n` | Your env var has a trailing newline, Google rejects OAuth with "The OAuth client was not found", you reflash the env twice trying to find the typo |
| D | IAP OAuth Admin API was shut down 2026-03-19 | `gcloud alpha iap oauth-brands list` errors with no explanation, you spend 45 minutes looking for the new SDK path — there is none |

This script knows all four.

## Install

```bash
git clone https://github.com/pusheeen/tools ~/pusheeen-tools
chmod +x ~/pusheeen-tools/google-oauth-setup/bin/google-oauth-setup
# optional: symlink onto PATH
ln -s ~/pusheeen-tools/google-oauth-setup/bin/google-oauth-setup ~/.local/bin/
```

## Prereqs

- **gstack browse** — the headless browser we use to drive Google Cloud Console
  (https://github.com/garry-tan/gstack)
- **Chrome** installed with an active Google session
- **jq** (`brew install jq`)
- **Vercel CLI** (`npm i -g vercel`) — only if passing `--vercel-project`

## Usage

```bash
google-oauth-setup \
  --app-name "Loop" \
  --redirect-uri "https://loop.oddgroup.ai/api/auth/google/callback" \
  --project-id my-gcp-project-id \
  --vercel-project loop-pm-agent \
  --env-prefix GOOGLE
```

Flags:
- `--app-name` (required) — name shown on Google's consent screen
- `--redirect-uri` (required) — where Google sends the auth code
- `--project-id` (required) — your GCP project ID
- `--vercel-project` (optional) — if set, the tool writes `${PREFIX}_CLIENT_ID`
  and `${PREFIX}_CLIENT_SECRET` to Vercel across prod/preview/dev using
  `printf` (not `echo`, because of Gotcha C)
- `--env-prefix` (default `GOOGLE`) — env var name prefix

## What happens

1. **Imports Google cookies** from Chrome using `.google.com` (Gotcha A)
2. **Opens GCP Credentials page** in the headless browser
3. **Prints a runbook** for the manual clicks (create OAuth client,
   configure redirect URI) — prompts you to press Enter when the
   success modal is visible
4. **Scrapes Client ID + Client Secret** from the DOM of the success
   modal (Gotcha B — regex `GOCSPX-[A-Za-z0-9_-]+` catches the secret
   while it's in the DOM)
5. **Sets Vercel env vars** correctly using `printf '%s'` (Gotcha C)

## Limitations

- Requires you to already have a GCP project created. The script does
  not create GCP projects (that's a different ball of wax: billing,
  API enablement, org policies).
- Requires you to be logged into Google in Chrome with a session that
  has OAuth admin rights on the GCP project.
- The success modal must be open in the browse tool window when you
  press Enter at step 3. If you accidentally close it, Google hides
  the secret forever — you have to delete the client and re-create it.

## Manual fallback

If the script fails at any step, every step is documented in the code
with a comment explaining the gotcha. You can run the pieces manually:

```bash
# Step 1: import cookies
~/.claude/skills/gstack/cookie-import-browser chrome --domain .google.com

# Step 4: scrape credentials (after you've clicked CREATE in the browser)
~/.claude/skills/gstack/browse/dist/browse js '(() => {
  const html = document.documentElement.outerHTML;
  return JSON.stringify({
    ids: [...new Set((html.match(/[0-9]+-[a-z0-9]+\.apps\.googleusercontent\.com/g) || []))],
    secrets: [...new Set((html.match(/GOCSPX-[A-Za-z0-9_-]+/g) || []))]
  });
})()'

# Step 5: set Vercel env without trailing newlines
printf '%s' "$CLIENT_ID"     | vercel env add GOOGLE_CLIENT_ID     production
printf '%s' "$CLIENT_SECRET" | vercel env add GOOGLE_CLIENT_SECRET production
```

## Proven with

- `loop.oddgroup.ai` (2026-04-23) — shaved future setup time from
  ~2 hours to ~5 minutes
