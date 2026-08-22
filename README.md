# Cupping Log

A private record of coffee drinks by shop. No accounts, no sharing, no server. Ratings live in your browser's local storage; the only backup is the JSON file you export.

Three files, no build step: `index.html`, `manifest.json`, `icon.svg`.

---

## 1. Get a Google Maps key

The app uses Google Maps to draw the map and Google Places to find shops. Google's terms require Places results to be shown on a Google map, so these go together.

1. Go to `console.cloud.google.com`, create a project, and enable billing. Billing is mandatory even inside the free allowances.
2. Under **APIs & Services → Library**, enable exactly two APIs:
   - **Maps JavaScript API**
   - **Places API (New)** — not the legacy "Places API"
3. **Credentials → Create credentials → API key.**
4. Restrict the key immediately:
   - *Application restrictions* → **Websites** → add `https://YOURNAME.github.io/*`. Add `http://localhost:*` too if you'll test locally.
   - *API restrictions* → **Restrict key** → select only the two APIs above.
5. Under **APIs & Services → each API → Quotas**, set a low daily cap (e.g. 200 requests/day). This is the real protection against a surprise bill.

### What this will cost you

Nothing, in practice. The old $200 universal credit was retired in March 2025 and replaced with per-SKU monthly allowances — roughly 10,000 free calls/month for Essentials SKUs and 5,000 for Pro. A place search bills through Text Search Pro at about $32 per 1,000 calls *past* the free allowance. Personal use is on the order of 100 calls a month.

The app requests only `id`, `displayName`, `formattedAddress`, and `location`. It deliberately does **not** request Google's own `rating` or `reviews` fields, which would push each call into a more expensive SKU tier — and you're recording your own scores anyway.

---

## 2. Put it on GitHub Pages

```bash
git init
git add index.html manifest.json icon.svg README.md
git commit -m "Cupping Log"
git branch -M main
git remote add origin https://github.com/YOURNAME/cupping-log.git
git push -u origin main
```

Then **Settings → Pages → Source: Deploy from a branch → main / (root)**. Live at `https://YOURNAME.github.io/cupping-log/` within a minute or two.

**Your API key is not in this repository.** You paste it into the app's Settings tab once, and it's stored in local storage on that machine. Never hardcode it into `index.html` — public repos get scraped for keys within days.

---

## 3. Make it a dock app on macOS

- **Chrome:** open the Pages URL → ⋮ menu → *Cast, save and share* → *Install page as app*.
- **Safari:** open the URL → *File → Add to Dock*.

Either produces a standalone window with its own dock icon. Geolocation works because Pages serves over HTTPS.

---

## How it works

**Ratings.** Every visit is its own record — order a cortado at the same shop five times and you get five entries. Scores are 0–10 to one decimal.

**Rankings.** Pick a drink, get shops ranked by your mean for that drink, with *n* and sample standard deviation shown.

There's a second ranking mode, **shrunk mean**, because ranking by raw mean rewards small samples: one lucky 9.4 outranks a shop you've visited eight times averaging 8.9. The shrunk estimate pulls each shop toward the pooled mean in proportion to how little data supports it:

```
x̄* = (m·μ + Σx) / (m + n)
```

with prior weight `m = 2` and `μ` the pooled mean across all ratings for that drink. It's a crude empirical-Bayes shrinkage — a fixed prior weight rather than one estimated from between-shop variance — but for ordering a personal list it does the job. Change `PRIOR_WEIGHT` in `index.html` if you want it stronger or weaker.

**Notes.** Per-shop notes autosave as you type. Each individual rating can also carry its own note.

---

## Your data

Local storage is durable but not permanent. Clearing site data, switching browsers, or a new Mac all lose it.

**Export from the Settings tab regularly.** Import merges rather than replaces: new ratings are added by ID, and shop records are overwritten by the imported version, so a stale export will overwrite newer shop notes. Ratings are safe either way.

The export format:

```json
{
  "exportedAt": "2026-08-22T14:00:00.000Z",
  "version": 1,
  "shops":   { "<google_place_id>": { "id","name","address","lat","lng","notes","addedAt" } },
  "ratings": [ { "id","shopId","drink","score","date","note" } ]
}
```

Flat and trivially parseable — read it straight into pandas or R if you want to do anything real with it.

## Swapping pieces later

Two seams in `index.html` are deliberately isolated:

- **`Store`** — four functions wrapping local storage. Point them at Supabase, IndexedDB, or a GitHub-repo JSON file and nothing else changes.
- **`Places`** — two methods returning `{id, name, address, lat, lng}`. Any provider matching that shape drops in. Note the terms constraint: non-Google POI data must not be drawn on a Google map, so swapping the provider means swapping the map too.
