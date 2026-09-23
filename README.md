# demo-login

A single-file proof-of-concept landing page for **redirect and unintended-login
findings**. Host it anywhere static, point a vulnerable redirect at it, and the
page renders the evidence you need for the report.

Built for the moment a target application hands control of its navigation to a
URL you supply — `returnUrl`, `next`, `continue`, `RelayState`, an OAuth
`redirect_uri`, a framed login — and you need to show where the user actually
ended up.

> **Authorised testing only.** Use this against applications you own or hold
> written permission to test. The page is deliberately generic and imitates no
> real organisation; keep it that way.

## Why a static page

There is no server and no build. The point is to have a URL you can paste into
a parameter, so it runs from anywhere that serves a file:

* **GitHub Pages** — enable Pages on this repo, use `https://<user>.github.io/demo-login/`
* **A gist** — raw gist URLs work, though they serve as `text/plain` on
  gist.githubusercontent.com; for a rendered page prefer Pages or any static host
* **Locally** — `python3 -m http.server` while you work

## What it shows

On landing, the evidence panel records:

| Field | Why it matters |
|---|---|
| **Evidence ID** | short random id, so a screenshot ties back to a specific run |
| **Landed at** | UTC timestamp |
| **Landing URL** | the full URL the browser ended on — the redirect actually happened |
| **Referrer** | which page sent the user here — the strongest single artefact |
| **Query parameters** | whatever the redirect carried across |
| **Context** | flags an iframe, and flags `window.opener` being set |

Submitting the form adds a **Form captured** row showing the username and the
password's length. Nothing leaves the browser — that row is the demonstration
that a credential form on an attacker-controlled origin would have taken them.

### About an empty Referrer

A cross-origin `302` frequently arrives with no referrer at all, because the
origin's `Referrer-Policy` strips it. The page says so explicitly rather than
showing a blank, since a missing referrer is normal redirect behaviour and does
not weaken the finding — the landing URL and the screenshot carry it.

### Context flags

`window.opener is set` means the page that opened this one is reachable via
`opener.location`, i.e. reverse tabnabbing is available — worth noting whenever
a target opens outbound links without `rel="noopener"`.

## Optional out-of-band record

Add `?c=<collector-host>` to log a landing even when nobody is watching:

```
https://your-host.example/?c=abc123.oast.fun
```

The page requests `https://<collector>/landed?id=<evidence-id>&ref=<referrer>`,
which shows up in Interactsh or any OAST listener. Useful for stored or delayed
redirects that fire long after you have moved on.

`c` is validated as a bare hostname and always fetched over HTTPS, so a hosted
copy of this page cannot be turned into a request gadget pointed at a URL of
someone else's choosing.

## Reporting with it

1. Host the page.
2. Put its URL in the vulnerable parameter.
3. Screenshot the result — the login form and the evidence panel are laid out
   to fit one capture.
4. Quote the Evidence ID and Landing URL in the write-up.

The screenshot makes the impact self-evident: the victim followed a link on the
target's own domain and arrived at a credential form on somebody else's.
