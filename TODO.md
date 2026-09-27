# TODO: Migrate from static NGINX to Rust/Axum service

Current state: a handful of static HTML files (`index.html`, `love/index.html`, `static/*.webp`) served directly by NGINX. Goal: replace NGINX's role with a Rust binary built on Axum, while keeping (or intentionally replacing) the behaviors NGINX currently gives us for free — static file serving, case-insensitive `/love` routing, TLS, caching headers, etc.

Production requirements:
- [ ] Site must be served on the domain `https://jonathanmccormickjr.com/`.
- [ ] Any HTTP request must automatically redirect to HTTPS (no plain-HTTP responses served).
- [ ] The base page template needs an overhaul so styling/layout/nav are consistent and intuitive across all pages, not just `index.html` and `love/index.html`.
- [ ] The entire site (every page, not just `/love`) must be mobile friendly.

## 1. Project scaffolding
- [ ] `cargo init --name jonathanmccormickjr` in the repo root.
- [ ] Add core deps: `axum`, `tokio` (`full` or `rt-multi-thread` + `macros`), `tower`, `tower-http` (`fs`, `compression-full`, `trace`, `set-header`).
- [ ] Add `tracing` + `tracing-subscriber` for structured logging (replacing NGINX access/error logs).
- [ ] Decide on a template engine for the HTML (`askama` or `minijinja`) vs. continuing to serve raw HTML strings — recommend `askama` for compile-time-checked templates.

## 2. Routing parity with current NGINX behavior
- [ ] `GET /` → serve home page (currently `index.html`).
- [ ] `GET /love`, `/love/`, `/LOVE`, and other case variants → serve the love page. Implement via a small case-insensitive path-normalizing middleware (or explicit route list) since Axum's router is case-sensitive by default.
- [ ] Serve `/static/*` assets (`book.webp`, `bowtie.webp`, `coca-cola.webp`, `cute-cat.webp`, `kiss.webp`, `mirror.webp`, `red.webp`) via `tower_http::services::ServeDir`, preserving correct `Content-Type` and far-future `Cache-Control` headers for immutable assets.
- [ ] 404 handler for unmatched routes (custom not-found page, or a minimal one to start).
- [ ] Decide on `www` vs non-`www` behavior for `https://jonathanmccormickjr.com/`, and implement the mandatory `http` → `https` redirect (see Production requirements above) — either in Axum middleware or via a thin NGINX/Caddy layer in front purely for TLS + redirects.

## 3. Port page content into templates
- [ ] Extract shared `<head>`/header/footer markup from `index.html` and `love/index.html` into a base template to avoid duplicating the CSS reset, nav link, etc.
- [ ] Overhaul that base template's styling/layout/nav so every page looks and behaves consistently and intuitively (see Production requirements above), and audit it for mobile friendliness (responsive layout, viewport meta tag, touch-friendly nav) across all breakpoints.
- [ ] Move the client-side age calculation (`love/index.html` `<script>`) to the server: compute age from DOB in Rust at request/render time instead of shipping JS to the browser.
- [ ] Confirm no other client-side behavior needs to move server-side.

## 4. Content update — `/love` page rewrite
- [ ] Rework "About Me" section to lead with who I am (values, lifestyle, what I'm building) in a way that's easy to skim.
- [ ] Add/expand a section describing what everyday shared life together would actually look like day to day (routines, home life, partnership style) — not just abstract values.
- [ ] Add a "big 3" wishes section, Tashiro (*The Science of Happily Ever After*) style — content pending; see chat for the three specific wishes to write in.
- [ ] Keep the existing Family Vision table and "How I Will Treat You" section, integrating new copy around them rather than duplicating.
- [ ] Re-check mobile responsiveness (as part of the site-wide mobile-friendly requirement) and image alt text after content changes.

## 5. Headers, security, and production hardening
- [ ] Add `tower_http::set_header` middleware for security headers (`X-Content-Type-Options`, `Referrer-Policy`, `Content-Security-Policy`, etc.) that NGINX may currently be setting (or not) — audit current live NGINX config on the server for anything to carry over before decommissioning it.
- [ ] Add `tower_http::compression::CompressionLayer` (gzip/br) for HTML/CSS/text responses.
- [ ] Add `tower_http::trace::TraceLayer` for request logging.
- [ ] Decide TLS termination strategy: Axum terminates TLS directly (`axum-server` + `rustls`) vs. keeping NGINX/Caddy/Cloudflare in front as a reverse proxy and only replacing what NGINX serves. Recommend keeping a thin reverse proxy for TLS/cert renewal unless there's a strong reason to do it in-process.
- [ ] Set up graceful shutdown (`axum::serve` + `tokio::signal`) so deploys don't drop in-flight requests.

## 6. Deployment
- [ ] Write a `Dockerfile` (multi-stage: `cargo build --release` in a builder stage, copy binary + `static/` into a slim runtime image).
- [ ] Or: write a `systemd` unit file if deploying the binary directly on the existing host.
- [ ] Update DNS/reverse-proxy config for `jonathanmccormickjr.com` to point at the new Axum service's port instead of NGINX's static root, keeping HTTPS on `https://jonathanmccormickjr.com/` and the HTTP→HTTPS redirect intact.
- [ ] Decommission the old NGINX static-file config once the Axum service is verified working end-to-end (keep NGINX only if it's still needed for TLS termination per the decision in section 5).

## 7. Testing & CI
- [ ] Add integration tests (via `axum::body` + `tower::ServiceExt::oneshot`, or `reqwest` against a spawned test server) covering: `/`, `/love` and case variants, static asset retrieval, 404 behavior.
- [ ] Add a CI workflow (GitHub Actions) to run `cargo fmt --check`, `cargo clippy`, `cargo test` on push.
- [ ] Add a build/release step to CI to produce the deployable artifact (binary or Docker image).

## 8. Docs
- [ ] Update `README.md` to describe the new Rust/Axum architecture, local dev instructions (`cargo run`), and deployment steps, replacing any stale NGINX-era notes.
