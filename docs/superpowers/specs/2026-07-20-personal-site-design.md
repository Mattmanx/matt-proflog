# Personal Site (mehalso.com) — Design Spec

- **Date:** 2026-07-20
- **Status:** Draft for review
- **Owner:** Matt Mehalso

## 1. Context & Goals

A personal website that serves as a professional brand independent of LinkedIn, hosting
a resume/CV and an occasional blog for short essays and musings. It doubles as a
deliberate learning vehicle for technologies outside the author's day-to-day work.

The author is a senior director / architect in financial services with deep Java/Spring
Boot, Oracle, and on-prem + OpenShift/Kubernetes experience. The learning target is the
*complementary* territory: modern front-end, cloud-native/container hosting, managed
Postgres, Infrastructure-as-Code, and modern auth.

**Primary goal:** stand up a credible personal brand + blog, and get it live early.

**Secondary goal:** maximize hands-on learning of cloud/containers/IaC, lightweight
backend service design, modern auth, and modern front-end — *without* letting that
learning surface put the public site's uptime or maintainability at risk.

**Budget:** ~$5–20/month.

## 2. Guiding Principle

**Content is the product, and the learning surface is isolated from the serving path.**

- The public site is static and has **no runtime dependency** on any backend or database.
  It survives neglect.
- The backend (the learning playground) is a **separate, blast-radius-isolated** service.
  It can break, be rebuilt, or be redeployed on different infra without ever affecting
  the public site.
- Content durability is entrusted to **git/GitHub** (versioned, backed up, free), not to a
  database the author must operate long-term.

## 3. Decisions Summary

| Concern | Decision | Rationale |
|---|---|---|
| Content model | Hybrid: static public site; **git/GitHub is the source of truth** for content | Durable, versioned, free; public path needs no DB; isolates learning from serving |
| Public front-end | **Astro** (markdown/MDX, Tailwind, static) | Content-first, zero-JS default, great SEO/perf, gentle on-ramp; React islands where needed |
| Admin front-end | **React SPA** | Highest-leverage modern FE skill; rich editor ecosystem |
| Backend | **Spring Boot** authoring API, containerized | Plays to author's strength while the *deployment* is the new learning |
| Backend role | **Git-writing authoring API**: commits markdown to the content repo | Keeps git as system of record; DB is not on content critical path |
| Database | **Neon** serverless Postgres | Scales to zero, cheap, standard Postgres (portable), holds only operational data |
| Hosting model | **Portable-first**: Dockerfile + Terraform + GitHub Actions; deploy to **Google Cloud Run** initially | IaC is the durable, transferable skill; cloud becomes a thin swappable layer; Cloud Run scales to zero (budget) |
| Static hosting | **Cloudflare Pages** (GitHub Pages as fallback) | Free, global CDN, git-connected, trivially portable |
| Auth | **Login with GitHub (OIDC/OAuth2)**, Spring Security OAuth2 client, **BFF session** | Modern standards-based learning; no passwords stored; thematically clean with git content |
| Design direction | **Content-forward, refined restraint** — leerob.io as north star | Author is not a front-end developer; the site should serve content, then disappear |

## 4. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  PUBLIC PATH (always up, no DB, no backend on critical path) │
│                                                              │
│  GitHub content repo (.md/.mdx = source of truth)            │
│        │ push                                                │
│        ▼                                                     │
│  GitHub Actions → Astro build → Cloudflare Pages (CDN)       │
│        ▲                                                     │
│        │ visitors read static HTML                           │
└────────┼─────────────────────────────────────────────────────┘
         │ commits content
┌────────┼─────────────────────────────────────────────────────┐
│  LEARNING / AUTHORING PATH (isolated blast radius)           │
│        │                                                     │
│  Spring Boot authoring API  ── OAuth2 "Login with GitHub"    │
│   (Docker → Cloud Run,        (BFF: httpOnly session cookie) │
│    scale-to-zero)                                            │
│        │                    ── Neon Postgres                 │
│        │                       (users/roles/drafts/audit)    │
│        ▼ commits markdown via scoped GitHub App token        │
│  GitHub content repo ─────────────────────────────┘          │
│        ▲                                                     │
│  React admin SPA (behind auth) — live markdown editor        │
│                                                              │
│  All infrastructure described in Terraform.                  │
└─────────────────────────────────────────────────────────────┘
```

### 4.1 Repositories

- **Code repo** (`matt-proflog`, this repo): Astro site, React admin, Spring Boot backend,
  Terraform, CI/CD.
- **Content repo** (separate, e.g. `mehalso-content`): markdown/MDX posts, resume data,
  images. Kept separate so the authoring token has `contents:write` on *content only* and
  **cannot** touch application code or CI configuration (closes the repo-write →
  arbitrary-CI-execution escalation path).

*(Open item for implementation: whether content is a separate repo or an isolated
directory in a monorepo with a path-scoped token. Default: separate repo.)*

### 4.2 Data flows

**Publish (via admin UI):**
1. User signs in to the React admin via "Login with GitHub" (OIDC).
2. User writes/edits a post; drafts autosave to Postgres.
3. On **Publish**, the Spring Boot API commits the markdown file to the content repo using
   a **scoped, short-lived GitHub App installation token**.
4. The commit triggers GitHub Actions → Astro rebuild → deploy to Cloudflare Pages.

**Publish (fallback, always available):** author commits markdown to the content repo
directly. No backend required. This is the Phase 1 workflow and the permanent safety net.

**Public read:** visitor requests a page → served as static HTML from the CDN. Zero backend
or DB involvement.

## 5. Component Specifications

### 5.1 Public site (Astro)
- **Purpose:** render resume/CV, blog index, individual posts, and standalone pages (about,
  now, etc.) as static HTML.
- **Interface:** consumes markdown/MDX + structured resume data from the content repo at
  **build time**. Emits static assets.
- **Depends on:** content repo, Tailwind, build via GitHub Actions.
- **Notes:** RSS feed, sitemap, and OpenGraph/SEO tags generated at build. React islands
  only where genuine interactivity is needed.

### 5.2 Content store (GitHub)
- **Purpose:** durable, versioned system of record for all content.
- **Interface:** git. Posts are markdown/MDX with YAML frontmatter; resume is a structured
  data file (YAML/JSON) rendered by Astro.
- **Post frontmatter (initial):** `title`, `slug`, `date`, `updated`, `summary`, `tags`,
  `author`, `draft`.
- **Depends on:** nothing at runtime.

### 5.3 Authoring backend (Spring Boot)
- **Purpose:** authenticated API to create/edit/publish content and manage drafts.
- **Interface:** REST API consumed by the React admin. Commits markdown to the content repo
  via a GitHub App token. Session established via BFF (httpOnly cookie).
- **Depends on:** Neon Postgres (operational data), GitHub App credentials, GitHub OAuth.
- **Deployment:** Docker image → Cloud Run (scale-to-zero). No cloud-specific APIs in app code.

### 5.4 Auth
- **Purpose:** authenticate the operator(s); authorize actions.
- **Mechanism:** GitHub OIDC/OAuth2. Spring Security as OAuth2 client. **BFF pattern**: the
  backend holds tokens; the SPA holds only an httpOnly session cookie.
- **Authorization:** a `users` table + role model + allowlist of permitted GitHub accounts.
  v1 has one row (owner). See §6.

### 5.5 Admin UI (React SPA)
- **Purpose:** authoring experience — list/create/edit/publish posts, live markdown editor
  with preview, draft management.
- **Interface:** calls the backend API; relies on the BFF session cookie.
- **Depends on:** backend API. Built around "the current signed-in user's" content.

### 5.6 Infrastructure & CI/CD
- **Terraform:** describes Cloud Run service, Neon, DNS, secrets, Cloudflare Pages project.
  The headline portable skill; makes an AWS/Azure move a rewrite of the provider layer only.
- **GitHub Actions:** builds/tests the backend image, runs Terraform, builds + deploys the
  Astro site, deploys the admin SPA.

## 6. Multi-User Extensibility (design-for, don't-build)

A future may add family members as additional users with undefined functionality. To avoid
painting into a corner **without building multi-user features now**:

- Real `users` table keyed on GitHub identity (`sub`/id), with a **role** field
  (`OWNER`, `AUTHOR`, …) and an **allowlist** gating who may authenticate into the admin.
- Authorization is role/allowlist-based, **never** a hardcoded "is this me?" check.
- Posts carry an `author` attribution field.
- Admin UI scopes content to the current signed-in user.

**Explicitly out of scope for v1:** user-management/invitation UI, granular per-user
permissions, and any family-facing features. Adding a user later = adding an allowlist row.

## 7. Design / UX Direction

North star: **leerob.io** — content-forward, unpretentious, refined restraint. The site's
job is to make the writing and the author's credibility legible, then get out of the way.

- Simple, direct layout; **inlined prose links**; clean sans-serif typography.
- Restrained near-monochrome palette with one subtle accent.
- Fast; minimal gratuitous motion.
- **Not** a front-end showcase (author is not a front-end developer) and **not** a
  retro/90s artifact.

Detailed visual decisions (exact fonts, accent color, dark-mode default, spacing scale) are
deferred to implementation, explored via the frontend-design skill and clickable mockup
Artifacts before code.

## 8. Security Considerations

- **Authoring token:** GitHub App installation token — short-lived (~1h, auto-refreshed),
  single repo, `contents:write` only. Worst-case leak = revertable junk commits to the
  content repo; cannot reach code or CI. Never exposed to the browser (server-side only).
- **Repo separation:** content repo separate from code repo removes the repo-write →
  CI-execution escalation path.
- **Auth:** BFF session (httpOnly cookie) avoids storing tokens in browser JS (XSS
  mitigation). Allowlist restricts who can authenticate.
- **Secrets:** managed via cloud secret manager + Terraform; never committed.

## 9. Out of Scope (v1 / YAGNI)

Comments, contact-form backend (email/social links only), managed Kubernetes, heavy CMS
features, multi-user management UI, analytics beyond a privacy-friendly counter (optional
in polish phase).

## 10. Project Plan / Phasing

Because git is the source of truth, **the backend is never required to publish** — so the
public site launches first and the backend is layered on as an isolated learning project.

- **Phase 1 — Public site MVP + soft launch.** Astro site (resume + a few hand-authored
  markdown posts), Tailwind, RSS/sitemap/SEO basics, GitHub Actions → Cloudflare Pages.
  Live on the free `*.pages.dev` URL with HTTPS. *Primary goal met early; author publishes
  by git commit.*
- **Phase 2 — Cloud / IaC foundations.** Terraform skeleton; "hello world" container on
  Cloud Run; Neon provisioned; secrets wiring. Low-stakes infra learning.
- **Phase 3 — Authoring backend.** Spring Boot API, GitHub OAuth login (BFF session),
  commit-on-publish via GitHub App token, drafts in Postgres, `users`/roles/allowlist.
- **Phase 4 — React admin UI.** Post list/create/edit/publish, live markdown editor +
  preview, draft management, wired to the API.
- **Phase 5 — Polish.** Visual refinement via mockups, dark mode, OpenGraph images,
  privacy-friendly analytics, performance/SEO pass.
- **Phase 6 — Connect custom domain (final).** Cut `www.mehalso.com` over via GoDaddy DNS →
  Cloudflare Pages with managed TLS. Done last so all churn happens off a throwaway URL and
  the real domain flips only when polished. (Existing GoDaddy-managed domain.)

## 11. Open Items (deferred to implementation)

- Content in a separate repo vs. path-scoped directory in the monorepo (default: separate repo).
- Cloudflare Pages vs. GitHub Pages for static hosting (default: Cloudflare Pages).
- Exact resume data format (YAML vs. JSON) and whether to offer a downloadable PDF.
- Markdown editor component choice for the admin.
- Whether the admin SPA is served by Cloud Run or as a separate static bundle behind auth.
