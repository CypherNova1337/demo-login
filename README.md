# demo-login

A single-file landing page that turns "the redirect goes somewhere I control"
into a screenshot you can put in a report.

Host it, point a vulnerable redirect at it, and it records where the browser
landed, what sent it there, and what a credential form on that origin would
have collected. No server, no build, no dependencies.

> **Authorised testing only.** Use it against applications you own or hold
> written permission to test. The page imitates no real organisation — keep it
> that way unless a signed scope says otherwise.

---

## Quick start

1. **Settings → Pages → Build and deployment → Source: GitHub Actions** (once).
2. Push to `main`. The workflow publishes to
   `https://<user>.github.io/demo-login/`.
3. Put that URL in whatever parameter you are testing.
4. Follow the link in a browser and screenshot the result.

Working locally instead:

```bash
python3 -m http.server 8000     # then http://127.0.0.1:8000/
```

A gist will not work — `gist.githubusercontent.com` serves raw files as
`text/plain`, so you get markup instead of a rendered page.

---

## What it proves

| Finding | How to use the page | What the panel shows |
|---|---|---|
| **Open redirect** | `?returnUrl=`, `?next=`, `?url=`, `?dest=`, `?continue=`, `?redirect=` → your Pages URL | **Landing URL** on your origin, **Referrer** naming the target's page |
| **OAuth / OIDC `redirect_uri`** | Swap `redirect_uri` for your URL and follow the flow | **Query parameters** — a leaked `code` or `state` lands here in full |
| **SAML `RelayState`** | Set `RelayState` to your URL | Landing URL plus whatever the IdP forwarded |
| **Reverse tabnabbing** | Get the target to open your URL via `target="_blank"` | **Context** flags `window.opener is set` |
| **Framed login / clickjacking** | `<iframe src="https://target/...">` pointing at a redirect to your URL | **Context** flags `rendered inside an iframe` |
| **Post-auth redirect** | Set the post-login `next` parameter, then authenticate | Landing URL — the session lands off-origin |
| **Referer leakage** | Any redirect off the target | **Referrer** shows exactly what the target leaked |

The login form matters for all of them. Landing on a blank page proves a
redirect; landing on a working credential form is what turns it into an
account-takeover narrative a triager will act on.

---

## Reading the panel

| Field | Use in the write-up |
|---|---|
| **Evidence ID** | Random per load. Quote it so a screenshot ties to one specific test. |
| **Landed at** | UTC timestamp. |
| **Landing URL** | The full URL the browser ended on, including everything the redirect carried. |
| **Referrer** | The page that sent the browser here — the single strongest artefact. |
| **Query parameters** | Parsed out, so a leaked token is legible in the screenshot. |
| **Context** | Iframe and `window.opener` flags. |
| **Form captured** | Appears on submit: username, and the password's length only. |

**An empty Referrer is normal.** A cross-origin `302` usually arrives with none,
because the origin's `Referrer-Policy` strips it. The page says so rather than
showing a blank cell, so a reviewer doesn't read a gap as a failed PoC. The
landing URL carries the finding on its own.

**Nothing is transmitted.** The captured row is rendered from the browser's own
DOM. Submitting shows what *would* have been harvested without the page ever
collecting it — which is what you want when demonstrating against a real user's
session.

---

## Out-of-band logging

For redirects that fire later — a stored `returnUrl`, an email link, an admin
who opens it next Tuesday — add a collector:

```
https://<user>.github.io/demo-login/?c=abc123.oast.fun
```

The page requests `https://<collector>/landed?id=<evidence-id>&ref=<referrer>`,
which lands in Interactsh or any OAST listener. You get the hit even though
nobody was watching the screen.

`c` is validated as a bare hostname and always fetched over HTTPS, so a hosted
copy can't be turned into a request gadget aimed at a URL of someone else's
choosing.

---

## Worked example

Target accepts any absolute URL in `returnUrl`:

```
https://target.example/login?returnUrl=https://cyphernova1337.github.io/demo-login/
```

Open it, sign in (or just follow the redirect), screenshot.

Then write it up like this:

> **Open redirect → credential harvesting, `/login`**
>
> `returnUrl` is not validated against an allowlist, so authentication
> completes and the browser is sent to an arbitrary external origin.
>
> Reproduction: open the URL above. The browser lands on
> `cyphernova1337.github.io`, which serves a sign-in form. Evidence ID
> `MUDEP9NFED50`, referrer `https://target.example/login`.
>
> Impact: a link whose visible host is `target.example` delivers a user to a
> credential form on infrastructure the attacker controls, after the target's
> own login flow has vouched for the link.

The screenshot does the arguing — the form and the evidence panel fit one
capture.

---

## Customising

`index.html` is one file. The two things worth changing per engagement:

* **The heading** — "Project Portal" is deliberately generic. Match the client's
  own naming only when the signed scope covers it, and never a third party's
  brand.
* **The fields** — swap in whatever the real login asks for.

The evidence panel reads the form directly, so added fields are picked up
without touching the script.

### Hosting notes

A Pages site is public. The page sends `noindex` so it stays out of search
results, but treat the URL as reachable by anyone and keep client names out of
it. Take the site down, or make the repo private, when an engagement ends.
