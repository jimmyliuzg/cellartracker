# CellarTracker API Extension Notes

## Write Operations Limitation

CellarTracker uses **AWS WAF** (Web Application Firewall) with JavaScript challenges on all web endpoints (`/classic/*`, `/m/*`, `/purchasebulk.asp`, etc.).

### What works (no WAF)
- **Read/Export**: `GET xlquery.asp?User=...&Password=...&Table=...` — direct, WAF-free, works via Python requests
- **Hermes browser**: Full WAF challenge handling via real Chrome instance

### What's blocked (WAF-protected)
- Python `requests`, `urllib`, `curl` — no JS execution capability
- Playwright headless — detected as automation despite fingerprint spoofing
- Any raw HTTP client hitting `*.cellartracker.com` (except `xlquery.asp`)

### Workarounds
1. **Browser cookie extraction** — If we extract the WAF session cookie from a logged-in Hermes browser session, Python `requests` may be able to reuse it for subsequent writes
2. **Browser-as-API** — Use the Hermes browser tool set to navigate/click/type for write operations (proven working)
3. **Bulk CSV import** — `bulkfaq.asp` endpoint was not tested; may use a different auth path

## Mobile API Investigation

The mobile web interface uses clean RESTful URL patterns:
- `GET /m/my/cellar/add?q=<search>` — search wines
- `GET /m/wines/{iWine}/purchases/new` — add purchase form
- These pages are also WAF-protected

The native mobile app (iOS/Android) may use a separate API on `mobileapp.cellartracker.com` or a different backend. This was **not investigated** and could be a path to WAF-free writes.

## Reverse-Engineered URL Patterns

### Search / Add Flow
| Step | URL Pattern | Method |
|------|------------|--------|
| Search wines | `pickproducer.asp?szSearch=<query>&PickWine=on` | GET |
| Add to cart | `purchasebulk.asp?iCartNum0=1&iCart0=<iWine>` | GET |
| Purchase form | `purchasebulk.asp` (same URL, shows form fields) | GET |
| Save purchase | `purchasebulk.asp` (form POST) | POST |
| Edit purchase | `purchase.asp?iWine=<iWine>&iPurchase=<iPurchase>` | GET |
| Delete purchase | `purchase.asp` with `Delete=Delete+these+Bottles` | POST |
| Add tasting note | `editnote.asp` with note + rating + date | POST |

### Form Fields (Add Bottle)
| Field | Type | Description |
|-------|------|-------------|
| iQuant0 | number | Quantity |
| iSize0 | text | Bottle size (750ml, 1.5L, etc.) |
| Loc0 | text | Storage location |
| Bin0 | text | Bin identifier |
| Price0 | number | Cost per bottle |
| StoreName | text | Store purchased from |
| PurchaseDate | text | MM/DD/YYYY |
| Currency | text | USD, EUR, etc. |
| BottleNote | text | Optional note |
| Delivered | boolean | True=In My Cellar, False=Pending |

### Mobile UI URLs
| Pattern | Usage |
|---------|-------|
| `/m/wines/{iWine}/purchases/new` | Add purchase for specific vintage |
| `/m/wines/{iWine}/clone?Action=purchases%2Fnew` | New vintage from existing wine |
