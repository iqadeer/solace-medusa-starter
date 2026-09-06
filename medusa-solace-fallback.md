# Medusa + Solace — Fallback Plan

> **Status: PAUSED**
>
> Medusa + Solace is not being developed further at this point.
>
> The current priority is to evaluate Saleor as the primary ecommerce platform.
>
> This document records exactly where we stopped and what remains to be done if Saleor does not meet the project's requirements.

---

## Why This Work Is Paused

The current Medusa + Solace implementation has reached a point where continued development is producing too many integration and data-related issues.

The decision is therefore:

1. Stop adding features to the Medusa + Solace implementation.
2. Preserve the current working state.
3. Evaluate Saleor independently.
4. Only return to Medusa + Solace if Saleor fails to meet the project's requirements.

This is **not a deletion or abandonment of the codebase**. It is a deliberate fallback point.

---

# Current State

The current application is running with the existing Medusa + Solace architecture.

Some issues were encountered and resolved during development, including:

- Hero image layout issues.
- Image URL handling between the frontend and Strapi/Solace.
- Missing blog featured images.
- Missing blog slugs.
- Next.js static generation failures caused by incomplete CMS data.
- Authentication/configuration issues between application components.

The current image URL handling has been fixed in several locations, but this should be refactored before further development.

---

# First Task If We Resume

## 1. Centralize Strapi/Solace Image URLs

Image URL construction is currently duplicated in multiple components.

Do **not** continue adding:

```ts
;`${process.env.NEXT_PUBLIC_STRAPI_URL}${image.url}`
```

throughout the application.

Create a single utility, for example:

```text
src/lib/strapi/image-url.ts
```

or:

```text
src/utils/get-strapi-image-url.ts
```

The utility should:

- Accept a Strapi media object or image URL.
- Handle absolute URLs.
- Handle relative Strapi URLs.
- Use the configured Strapi base URL.
- Return a predictable value.
- Avoid duplicating environment-variable access throughout React components.

Example conceptual API:

```ts
getStrapiImageUrl(image)
```

Then components should use:

```ts
src={getStrapiImageUrl(post.FeaturedImage)}
```

rather than constructing URLs themselves.

### Also review

While doing this refactor, identify other places where Strapi-specific assumptions are duplicated.

The goal is to keep CMS integration logic in one place.

---

# If Medusa + Solace Is Resumed

After the image URL refactor, continue in the following order.

## 2. Create a Proper Solace Docker Compose Setup

Create a dedicated Solace Compose configuration.

The Compose setup should clearly define:

- Solace
- Medusa
- PostgreSQL
- Valkey/Redis if required
- Any Solace worker/background services
- Required environment variables
- Persistent volumes
- Network configuration

Where possible, reuse the existing PostgreSQL infrastructure rather than introducing an unnecessary second database.

The Compose setup should be suitable for local development first.

---

# 3. Create a Solace Docker Image

Once the Compose setup is stable:

- Create a production-capable Solace Dockerfile.
- Install production dependencies only.
- Configure the application for non-development execution.
- Ensure environment configuration is externalized.
- Avoid baking secrets into the image.
- Make the image reproducible.
- Pin important dependency/base-image versions where appropriate.

The resulting image should be suitable for use by Jenkins and production deployment.

---

# 4. Jenkins Build Pipeline

Create Jenkins build scripts/pipelines for the three application components:

1. Medusa
2. Solace
3. Storefront

The pipeline should eventually:

```text
Git push
   ↓
Jenkins
   ↓
Checkout
   ↓
Install dependencies
   ↓
Lint / type-check / tests
   ↓
Build Docker image
   ↓
Tag image
   ↓
Push image to registry
```

Images should use explicit version/build tags rather than relying exclusively on `latest`.

Example:

```text
medusa:<git-sha>
solace:<git-sha>
storefront:<git-sha>
```

A release tag can additionally produce a semantic version tag.

---

# 5. Jenkins Deployment Pipeline

After image builds are reliable, create deployment scripts.

The deployment flow should be approximately:

```text
Jenkins
   ↓
Build images
   ↓
Push registry
   ↓
Production server
   ↓
Pull new images
   ↓
Update Compose
   ↓
Restart/recreate services
   ↓
Health checks
   ↓
Verify deployment
```

Deployment should be repeatable and preferably reversible.

Keep production configuration separate from application source code.

---

# 6. Production Containers

Prepare the final production container architecture.

Expected application services:

```text
Nginx
  │
  ├── Storefront
  ├── Medusa
  └── Solace
```

Supporting services:

```text
PostgreSQL
Valkey/Redis
```

Background workers should run as separate Compose services where required.

Use production-specific:

- Environment variables
- Secrets
- Health checks
- Restart policies
- Resource limits where appropriate
- Persistent storage
- Logging configuration
- Networks

---

# 7. Nginx Configuration

Add production routing to the existing Nginx infrastructure.

The final configuration should provide appropriate domains/subdomains for:

- Storefront
- Medusa API
- Solace/admin interface if required

Example conceptual structure:

```text
Internet
   ↓
Nginx
   ├── shop.example.com
   │      ↓
   │   Storefront
   │
   ├── api.example.com
   │      ↓
   │   Medusa
   │
   └── admin.example.com
          ↓
       Solace
```

Configure:

- Reverse proxy
- TLS
- WebSocket support if required
- Appropriate proxy headers
- Request/body limits where required
- Timeouts
- Security headers
- Static asset handling where appropriate

Do not expose PostgreSQL or Valkey publicly.

---

# 8. Production Environment

Document the final production environment.

At minimum:

```text
Nginx
Medusa
Solace
Storefront
PostgreSQL
Valkey
```

Document:

- Ports
- Networks
- Volumes
- Domains
- Environment variables
- Secrets
- Image versions
- Backup requirements
- Deployment procedure
- Rollback procedure

---

# 9. CI/CD Completion

The final CI/CD flow should look approximately like:

```text
Developer
   ↓
Git
   ↓
Jenkins
   ↓
Tests
   ↓
Docker builds
   ↓
Container registry
   ↓
Production deployment
   ↓
Health checks
   ↓
Live application
```

The deployment should not require manually building containers on the production server.

---

# Important Decision

## Do Not Resume This Work Yet

The above tasks are the **fallback roadmap**, not the immediate development plan.

The current priority is:

> **Evaluate Saleor first.**

If Saleor provides a stable solution for the required ecommerce functionality, continue with Saleor and leave this stack paused.

If Saleor fails because of:

- Missing functionality
- Excessive customization requirements
- Integration limitations
- Operational complexity
- Performance/scaling problems
- Checkout/payment limitations
- CMS/content requirements
- Other unacceptable architectural constraints

then return here and continue from:

**Step 1 — Centralize Strapi/Solace image URL handling.**

---

# Resume Checklist

When returning to this project:

- [ ] Refactor Strapi image URL handling into one utility.
- [ ] Review other duplicated Strapi integration logic.
- [ ] Create Solace Docker Compose.
- [ ] Validate local Compose environment.
- [ ] Create production Solace Docker image.
- [ ] Create Jenkins build pipeline for Medusa.
- [ ] Create Jenkins build pipeline for Solace.
- [ ] Create Jenkins build pipeline for Storefront.
- [ ] Create Jenkins deployment pipeline.
- [ ] Prepare production Compose configuration.
- [ ] Configure production containers.
- [ ] Configure Nginx.
- [ ] Configure TLS/domains.
- [ ] Add health checks.
- [ ] Add backups/restore procedure.
- [ ] Add rollback procedure.
- [ ] Perform full production deployment test.

---

## Current Decision

**Medusa + Solace: PAUSED**

**Saleor: PRIMARY CANDIDATE**

**Resume point if Saleor fails: Step 1 — Image URL utility refactor.**
