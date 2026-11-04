# Deploy haively.com — Azure Static Web Apps (Free)

Do these steps in order. Each one depends on the one before it.

**Variables used below** (run once per terminal session, PowerShell users swap `export` for `$env:`):

```bash
export RG=rg-haively-web
export APP=swa-haively-web
export LOC=eastus2          # SWA regions: centralus, eastus2, eastasia, westeurope, westus2
```

---

## Phase 1 — Put the site in a repo

**Step 1.** Create a new GitHub repo named `haively-site` (separate from `haively-web`, which stays the portal).

**Step 2.** Copy this package into it. Layout must be:

```
haively-site/
├── .github/workflows/deploy.yml
├── site/
│   ├── index.html                  ← your exported design
│   ├── staticwebapp.config.json
│   └── (any images/fonts/css from the export)
└── DEPLOY.md
```

**Step 3.** Commit and push to `main`. The workflow will run and fail — that's expected; it has no token yet.

---

## Phase 2 — Create the Azure resource

**Step 4.** Sign in and pick the subscription:

```bash
az login
az account set --subscription "<your subscription name or id>"
```

**Step 5.** Create the resource group:

```bash
az group create --name $RG --location $LOC
```

**Step 6.** Create the Static Web App on the Free plan:

```bash
az staticwebapp create --name $APP --resource-group $RG --location $LOC --sku Free
```

**Step 7.** Print the default address and save it — you need it for DNS:

```bash
az staticwebapp show --name $APP --resource-group $RG --query defaultHostname -o tsv
```

Example output: `gentle-river-0a1b2c3d4.5.azurestaticapps.net`

**Step 8.** Get the deployment token:

```bash
az staticwebapp secrets list --name $APP --resource-group $RG --query properties.apiKey -o tsv
```

---

## Phase 3 — Connect GitHub to Azure

**Step 9.** In GitHub: repo → **Settings → Secrets and variables → Actions → New repository secret**
- Name: `AZURE_STATIC_WEB_APPS_API_TOKEN`
- Value: the token from Step 8

**Step 10.** Repo → **Actions → Deploy haively.com → Run workflow**. Wait for the green check.

**Step 11.** Open `https://<defaultHostname from Step 7>` and confirm the site loads.

---

## Phase 4 — Point haively.com at it

Do `www` first (simplest), then the bare domain.

**Step 12. www.haively.com** — at your DNS provider, add:

| Type  | Host | Value                         |
|-------|------|-------------------------------|
| CNAME | www  | `<defaultHostname from Step 7>` |

Then register it in Azure:

```bash
az staticwebapp hostname set --name $APP --resource-group $RG --hostname www.haively.com
```

**Step 13. haively.com (bare domain)** — start TXT validation:

```bash
az staticwebapp hostname set --name $APP --resource-group $RG \
  --hostname haively.com --validation-method dns-txt-token --no-wait
```

**Step 14.** Wait ~1 minute, then get the validation token:

```bash
az staticwebapp hostname show --name $APP --resource-group $RG \
  --hostname haively.com --query validationToken -o tsv
```

**Step 15.** At your DNS provider, add:

| Type | Host | Value                   |
|------|------|-------------------------|
| TXT  | @    | `<token from Step 14>`  |

**Step 16.** Point the bare domain at the app. Pick the row your DNS provider supports:

| Your DNS provider supports…                     | Add this record                                   |
|--------------------------------------------------|---------------------------------------------------|
| ALIAS / ANAME / CNAME flattening (Cloudflare, Namecheap, Route 53, DNSimple) | `ALIAS @ → <defaultHostname>` |
| None of those (e.g. GoDaddy, Squarespace)        | Use registrar **forwarding**: `haively.com → https://www.haively.com` (301), and skip Step 13–15 |
| Want to keep it all in Azure                     | Move DNS to an Azure DNS zone (small monthly fee) and add an alias record to the Static Web App |

**Step 17.** Check status until both show `Ready` (can take up to an hour; SSL certificates are issued automatically):

```bash
az staticwebapp hostname list --name $APP --resource-group $RG -o table
```

**Step 18.** In the Azure portal → your Static Web App → **Custom domains**, set your preferred one (`haively.com` or `www.haively.com`) as **default**.

---

## Phase 5 — Verify

**Step 19.** Confirm all three load over HTTPS with a valid padlock:
- `https://haively.com`
- `https://www.haively.com`
- `https://haively.com/some-page-that-does-not-exist` (should show the home page, not an Azure error)

**Step 20.** Confirm security headers are present:

```bash
curl -sI https://haively.com | grep -iE "strict-transport|x-frame|x-content-type|referrer"
```

---

## Updating the site later

Edit files in `site/`, commit, push to `main`. Live in ~1 minute.
Pull requests get their own preview URL automatically (Free plan: up to 3 at a time).

## When clients arrive — upgrade to Standard

Portal → Static Web App → **Settings → Hosting plan → Standard → Save**.
Same URL, same domain, no redeploy. Adds an SLA, more custom domains, and bandwidth overage instead of the site going offline at 100 GB/month.

## If a deploy fails

| Symptom | Fix |
|---|---|
| `deployment_token was not provided` | Secret name in Step 9 must match exactly |
| `Could not find index.html` | `index.html` must sit directly in `site/` |
| Custom domain stuck in `Validating` | Re-check the TXT record value; DNS can take up to 48h but usually minutes |
| Fonts/images missing after deploy | Asset paths in the HTML must be relative (`images/logo.svg`, not `C:\...` or `/Users/...`) |
