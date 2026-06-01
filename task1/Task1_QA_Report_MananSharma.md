# Task 1 — Web App QA & Debug Report

| Field | Details |
|-------|---------|
| **App Tested** | RealWorld "Conduit" — Angular Frontend |
| **Live Demo URL** | https://demo.realworld.show/ |
| **Source Repo** | https://github.com/realworld-apps/angular-realworld-example-app |
| **Date Tested** | June 2, 2026 |
| **Tester** | Manan Sharma |

---

## Bug Report Table

| # | Title / Summary | Steps to Reproduce | Expected vs Actual | Severity | Suspected Cause |
|---|----------------|--------------------|--------------------|----------|-----------------|
| 1 | **Missing Security Headers — No CSP, X-Frame-Options, X-Content-Type-Options, HSTS** | 1. Open DevTools → Network tab. 2. Load `https://demo.realworld.show/`. 3. Inspect response headers. Or run: `curl -sI https://demo.realworld.show/` | **Expected:** Response includes `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`. **Actual:** None present. Only `server: nginx/1.29.1` returned, leaking server version. | **Critical** | Nginx config not hardened. No `add_header` directives. `server_tokens off` not set. |
| 2 | **Page `<title>` is always "Conduit" — never updates on navigation** | 1. Navigate to Home (`/`) — note tab title. 2. Navigate to `/login`. 3. Navigate to `/article/how-to-learn-javascript-efficiently`. 4. All show "Conduit". | **Expected:** Title updates per route, e.g. "Sign In — Conduit". Critical for SEO and screen readers. **Actual:** Always static "Conduit". | **Medium** | Angular app doesn't use `Title` service. `<title>` in `index.html` is hardcoded. |
| 3 | **No Open Graph / meta description tags — social sharing broken** | 1. Copy any article URL. 2. Paste in Slack/Discord/Twitter. 3. Link preview is blank. 4. `curl -s ... \| grep "og:"` returns nothing. | **Expected:** Pages include `og:title`, `og:description`, `og:image`, `meta description`. **Actual:** Zero meta/OG tags present. | **Medium** | Client-side rendering limitation. Server sends same static `index.html` for all routes. No SSR. |
| 4 | **Tags cannot be added/removed when editing an article** | 1. Log in. 2. Create article with tags. 3. Edit the article. 4. Add/remove tags. 5. Publish — tags unchanged. | **Expected:** Tags should be editable and persist on save. **Actual:** Tag changes silently ignored. Confirmed: [GitHub Issue #343](https://github.com/realworld-apps/angular-realworld-example-app/issues/343). | **High** | Editor component doesn't bind modified `tagList` to the PUT request payload. |
| 5 | **Outdated Ionicons v2.0.1 from external CDN without SRI** | 1. View page source. 2. Find: `href="//code.ionicframework.com/ionicons/2.0.1/css/ionicons.min.css"`. 3. Note: v2.0.1 (current: 7.x), protocol-relative URL, no `integrity` attribute. | **Expected:** Current version, explicit HTTPS, SRI hashes, or self-hosted. **Actual:** Ancient v2.0.1, external CDN = single point of failure, no SRI = supply chain risk. | **Medium** | Legacy dependency from original RealWorld template, never updated. |
| 6 | **No client-side input validation on registration form** | 1. Go to `/register`. 2. Leave all fields empty → click Sign up. 3. Enter "notanemail" + "1" → click Sign up. Both submit. | **Expected:** Inline validation errors before API call. Email format check. Password requirements shown. **Actual:** Form submits with empty/invalid data. Errors only from server. | **Medium** | No Angular form validators (`Validators.required`, `Validators.email`) implemented. |
| 7 | **Non-existent user profile shows blank page — no error** | 1. Navigate to `/profile/nonexistentuser12345`. 2. Observe the page. | **Expected:** "User not found" message or 404 redirect. **Actual:** Profile layout loads but completely empty — no username, no bio, no feedback. | **Low** | Profile component silently catches API 404 and renders template with undefined data. |

---

## Root-Cause Analysis — Issue #1: Missing Security Headers

**What is happening:** When any page of the Conduit application is loaded, the server responds with HTTP headers that contain no security directives. Running `curl -sI https://demo.realworld.show/` reveals only basic headers: `server: nginx/1.29.1`, `content-type: text/html`, `cache-control: no-cache`, and standard date/length headers. There is no `Content-Security-Policy`, no `X-Frame-Options`, no `X-Content-Type-Options`, no `Strict-Transport-Security`, and no `Referrer-Policy`. The server also discloses its exact version (`nginx/1.29.1`).

**Why this happens:** The Nginx configuration file was set up to serve the Angular SPA's static files but was never hardened with security headers. This is a classic oversight in AI-assisted "vibe-coded" deployments where the priority is getting the app running rather than making it production-ready. Since Angular is a pure client-side SPA with no server middleware, security headers must be added at the Nginx reverse proxy level.

**Why it matters:** Without `X-Frame-Options: DENY`, the app can be embedded in iframes for clickjacking attacks. Without `Content-Security-Policy`, there's no browser-enforced restriction on scripts/resources, amplifying XSS vulnerabilities. Without `Strict-Transport-Security`, users visiting via HTTP are vulnerable to SSL-stripping MITM attacks. The server version leak gives attackers a specific target for known CVEs.

**How to fix it:** Add the following block to the Nginx `server` configuration:

```nginx
server {
    # Hide server version
    server_tokens off;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://code.ionicframework.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' https://raw.githubusercontent.com data:;" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
}
```

After adding, validate with `nginx -t` and reload with `nginx -s reload`. The CSP policy should be tested to ensure compatibility with the Ionicons CDN and inline styles. Implementation time: ~15 minutes.
