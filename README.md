
mist_full_clone_single — README
=================================

This is a single-file, no-deps Python script that copies a Mist org’s config into another org. It’s built for macOS and works anywhere you have Python 3.

The majority of this is vibecoded. 

---------------------------------
What it copies (today)
---------------------------------

Org-level
- RF Templates
- Network Templates
- Org WLANs
- WxLAN Rules
- WxLAN Tunnels
- Org Webhooks

Site-level (for each site)
- Site (auto-create by name if missing)
- Site Settings
- Site WLANs
- Site WxLAN Rules
- Site WxLAN Tunnels
- Site Webhooks

Everything above is round-tripped with field filtering (id, timestamps, etc.) so you can run it repeatedly without trashing anything.

---------------------------------
What it does NOT copy
---------------------------------

- Inventory / device claims (you must unclaim/claim between orgs)
- Users, roles, SSO/SAML, API tokens
- Floorplans/maps/zones (API doesn’t provide a clean export+import pair)
- Secrets: PPSK passphrases, OAuth secrets, private keys
- Historical analytics/SLE/metrics

---------------------------------
Requirements
---------------------------------

- macOS with Python 3.x (the python.org build is fine)
- Internet access to your Mist API region (e.g., https://api.ac2.mist.com)
- Two API tokens: one that can read the source org, one that can write the destination org

SSL on macOS
- If you see CERTIFICATE_VERIFY_FAILED, run Apple’s bundled script once:
  Finder → Applications → Python 3.x → Install Certificates.command
- If your environment does TLS interception, set CA_BUNDLE_PATH in the script to your PEM bundle.
- There’s also an INSECURE_SSL=True kill-switch, but don’t use it unless you absolutely have to.

---------------------------------
Quick start (macOS)
---------------------------------

1) Save the script.
2) Open it (Python IDLE Editor is plenty) and edit the “EDIT THESE” block:

API_HOST   = "https://api.ac2.mist.com"
TOKEN_SRC  = "YOUR_SOURCE_TOKEN"
TOKEN_DST  = "YOUR_DEST_TOKEN"
SRC_ORG_ID = "source-org-uuid"
DST_ORG_ID = "dest-org-uuid"
COPY_MODE  = "org-and-sites"  # or: "org-only", "sites-only"
# Optional selectors:
LIMIT_SITE_NAMES = []         # e.g. ["Home A","Home B"]
SRC_SITE_ID = ""              # use with DST_SITE_ID for ID->ID copy
DST_SITE_ID = ""
# SSL options:
CA_BUNDLE_PATH = ""
INSECURE_SSL   = False

3) Save it in IDLE and Run
   
---------------------------------
Common use cases
---------------------------------

Clone everything (org + all sites) from A → B
- Set COPY_MODE = "org-and-sites"
- Leave LIMIT_SITE_NAMES empty
- Run the script

Clone only org objects (no sites)
- Set COPY_MODE = "org-only"

Clone a subset of sites by name
- Set COPY_MODE = "org-and-sites" (or "sites-only")
- Set LIMIT_SITE_NAMES = ["Site One","Site Two"]

Clone a single site ID → ID
- Set COPY_MODE = "sites-only"
- Set SRC_SITE_ID and DST_SITE_ID (both required)
- Leave LIMIT_SITE_NAMES empty (ignored when IDs are set)

---------------------------------
How it decides to create vs update
---------------------------------

- For templates/WLANs/webhooks, it matches by name (or SSID for WLANs).
- It compares a trimmed payload (without id, timestamps, etc.).
- If different → PUT (update). If missing → POST (create). If equal → skip.
- It never calls DELETE.

This means you can run it repeatedly; it will bring the destination into line without wiping any unrelated objects.

---------------------------------
Rate limits and retries
---------------------------------

Mist’s REST API enforces rate limits. The script:
- Uses pagination (limit=100, page=1..)
- Retries with backoff on 429/502/503/504
- Keeps requests lean (no giant batch posts)

If you’re moving a huge org, let it run; it’ll throttle itself.

---------------------------------
Troubleshooting
---------------------------------

SSL errors (CERTIFICATE_VERIFY_FAILED)
- Run Install Certificates.command (python.org build).
- Or set CA_BUNDLE_PATH to a PEM bundle.
- As a last resort: INSECURE_SSL=True.

401 Unauthorized
- Token wrong scope or wrong org. Verify both tokens and org UUIDs.
- Check API_HOST is the correct region for both orgs.

404 Not Found
- Wrong region host (e.g., you used ac2 but your org lives elsewhere).
- A site or org ID is mistyped.

429 Too Many Requests
- Expected under load. The script backs off and continues. If you’re hammering a big org, run it during a quiet window.

Objects didn’t “take”
- Some nested references in network templates may rely on existing objects. The script orders org objects first, then sites. If you’ve got custom cross-refs, run it twice; on the second pass, updates will settle.

---------------------------------
Security notes
---------------------------------

- Tokens are literal strings in the script. Treat the file as sensitive.
- Don’t commit tokens to Git. If you must, use environment variables and os.environ (you can adapt the script easily).
- Never leave INSECURE_SSL=True in production.

---------------------------------
Extending the script
---------------------------------

If you want to add more endpoints:
- Follow the existing pattern: export → slim → import (name match + digest)
- Add to export_org()/import_org() or export_site()/import_site()
- Keep it idempotent and no deletes unless you truly need prune semantics

If you want inventory helpers (CSV export + post-claim re-site), ask and we’ll graft that in cleanly without third-party libs.

---------------------------------
FAQ
---------------------------------

Can it copy PPSKs?
- Not the secrets. You can import your own CSV/JSON of PPSKs to the destination org/sites afterward.

Can it copy floorplans?
- Not end-to-end. The public API doesn’t give a supported “download image → upload image” flow. You can export map metadata separately.

Does it require a VPN or inbound ports?
- No. It talks to the Mist API over HTTPS from wherever you run it.

---------------------------------
License / Warranty
---------------------------------

No license provided here. Use at your own risk. It does not delete on purpose to avoid foot-guns.
