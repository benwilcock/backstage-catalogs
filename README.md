# backstage-catalogs

A collection of Backstage / Red Hat Developer Hub (RHDH) software catalog entity
files, organised into two independently loadable catalogs:

| Catalog | Index file | Contents |
|---------|-----------|----------|
| **Enterprise OSS** | `enterprise-catalog-index.yaml` | Groups, Domains, Systems, Components and APIs for popular open-source platform tooling (Kubernetes, Kafka, Keycloak, ArgoCD, etc.) |
| **Parasol Insurance** | `parasol-catalog-index.yaml` | Fictional internal catalog for a large multinational insurer — useful for demos that need a realistic, industry-specific microservices landscape |

Each index points to a set of YAML files in its own subdirectory (`enterprise/` or
`parasol/`). You load whichever catalogs you need by adding entries to your
`app-config.yaml` and can comment out either entry to exclude it for a particular
demo environment.

---

## Repository layout

```
backstage-catalogs/
  enterprise/                               # Enterprise OSS catalog entities
    enterprise-catalog-foundations.override.yaml
    enterprise-catalog-data.override.yaml
    enterprise-catalog-platform.override.yaml
    enterprise-catalog-devops-security.override.yaml
    enterprise-catalog-ai-dataeng.override.yaml
    enterprise-catalog-frameworks.override.yaml
    enterprise-catalog-apis.override.yaml
  parasol/                                  # Parasol Insurance fictional catalog entities
    parasol-catalog-foundations.override.yaml
    parasol-catalog-claims.override.yaml
    parasol-catalog-underwriting.override.yaml
    parasol-catalog-policy.override.yaml
    parasol-catalog-personal-lines.override.yaml
    parasol-catalog-commercial-specialty.override.yaml
    parasol-catalog-life.override.yaml
    parasol-catalog-digital.override.yaml
    parasol-catalog-data-platform.override.yaml
    parasol-catalog-apis.override.yaml
  enterprise-catalog-index.yaml             # Location index — enterprise OSS only
  parasol-catalog-index.yaml                # Location index — Parasol Insurance only
```

---

## How to load these catalogs in Backstage / RHDH

You need to make three additions to your `app-config.yaml` (or
`app-config.local.yaml` if you use one for local overrides):

1. A **GitHub integration** so Backstage can authenticate and read files from
   `github.com`
2. A **backend reading allowlist** entry for `raw.githubusercontent.com` so the
   catalog processor is permitted to fetch the raw YAML content
3. One or two **catalog location** entries pointing at the index files in this repo

### 1. GitHub integration

Backstage fetches catalog files from GitHub using its GitHub integration. You need
at least a personal access token (classic, with `repo` or `read:org` scope) or a
GitHub App credential.

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}   # set this in your environment or .env file
```

If you use a GitHub App instead of (or in addition to) a token:

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
      apps:
        - appId: ${GITHUB_APP_ID}
          clientId: ${GITHUB_APP_CLIENT_ID}
          clientSecret: ${GITHUB_APP_CLIENT_SECRET}
          webhookSecret: ${GITHUB_APP_WEBHOOK_SECRET}
          privateKey: ${GITHUB_APP_PRIVATE_KEY}
```

> **Note for the Parasol catalog only:** The Parasol entities carry a
> `backstage.io/source-location` annotation pointing at a fictional
> `https://github.parasol.com` host. To suppress SCM authentication warnings in
> the RHDH UI, add a placeholder integration entry for that host:
>
> ```yaml
> integrations:
>   github:
>     - host: github.parasol.com
>       apiBaseUrl: https://github.parasol.com/api/v3
>       token: parasol-fictional-catalog-placeholder
> ```

### 2. Backend reading allowlist

Backstage's `PlaceholderProcessor` (used for `$text` substitutions in API entity
definitions) and the catalog fetcher both need explicit permission to read from
`raw.githubusercontent.com`. Add it to `backend.reading.allow`:

```yaml
backend:
  reading:
    allow:
      - host: raw.githubusercontent.com
```

If you already have a `reading.allow` list, just append that entry — do not
replace the existing entries.

### 3. Permissive catalog entity rules

The catalog files contain a mix of entity kinds (`Component`, `API`, `Location`,
`Domain`, `System`, `Group`, `Resource`, `User`, `Template`, etc.). You need a
permissive top-level rule so Backstage does not silently drop kinds it hasn't been
explicitly told to accept:

```yaml
catalog:
  rules:
    - allow:
        - Component
        - API
        - Location
        - Template
        - Domain
        - User
        - Group
        - System
        - Resource
        - Plugin
        - Package
```

### 4. Catalog location entries

Add one entry per catalog you want to load under `catalog.locations`. Use
`type: url` and point at the index file for that catalog. Each entry can be
commented out independently to exclude a catalog from a particular environment.

```yaml
catalog:
  locations:
    # Enterprise OSS catalog -- comment out to exclude from this environment
    - type: url
      target: https://github.com/benwilcock/backstage-catalogs/blob/main/enterprise-catalog-index.yaml
      rules:
        - allow: [Location]

    # Parasol Insurance fictional internal catalog -- comment out to exclude from this environment
    - type: url
      target: https://github.com/benwilcock/backstage-catalogs/blob/main/parasol-catalog-index.yaml
      rules:
        - allow: [Location]
```

> **Why `allow: [Location]` on the location entry?**
> The index files are `Location` entities. The top-level rule in step 3 allows
> all kinds globally, but per-location `rules` provide a tighter scope. Allowing
> `Location` here lets the index be ingested; the sub-files it points to are then
> fetched and their entity kinds are validated against the global rules.

---

## Complete minimal example

The following is a self-contained `app-config.yaml` snippet that adds both
catalogs to a vanilla Backstage instance. Merge it with your existing config —
do not replace it wholesale.

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
    # Required only for the Parasol Insurance catalog (suppresses SCM auth warnings)
    - host: github.parasol.com
      apiBaseUrl: https://github.parasol.com/api/v3
      token: parasol-fictional-catalog-placeholder

backend:
  reading:
    allow:
      - host: raw.githubusercontent.com

catalog:
  rules:
    - allow:
        - Component
        - API
        - Location
        - Template
        - Domain
        - User
        - Group
        - System
        - Resource
        - Plugin
        - Package

  locations:
    # Enterprise OSS catalog -- comment out to exclude
    - type: url
      target: https://github.com/benwilcock/backstage-catalogs/blob/main/enterprise-catalog-index.yaml
      rules:
        - allow: [Location]

    # Parasol Insurance fictional internal catalog -- comment out to exclude
    - type: url
      target: https://github.com/benwilcock/backstage-catalogs/blob/main/parasol-catalog-index.yaml
      rules:
        - allow: [Location]
```

---

## Refreshing the catalog

Backstage caches catalog entities and refreshes them on a schedule (default every
few minutes). After adding the location entries above and restarting, the entities
will be ingested on first startup. Subsequent pushes to this repository will be
picked up automatically on the next refresh cycle — no restart required.

To force an immediate refresh in the Backstage UI, navigate to a Location entity
in the catalog and click **Refresh**.

---

## Other catalog files in this repo

This repo also contains a small number of legacy catalog files that pre-date the
enterprise and Parasol catalogs:

| File | Description |
|------|-------------|
| `org-tanzu.yml` | Tanzu organisational structure (demo only) |
| `system-polyglot-demo-apps.yml` | Polyglot microservices demo system |
| `system-backstage-genai.yml` | Backstage GenAI system |
| `system-tanzu-application-platform.yml` | Tanzu Application Platform system |
| `system-tanzu-developer-portal.yml` | Tanzu Developer Portal system |
| `system-where-for-dinner.yml` | Where For Dinner demo system |
| `ai-asset-catalog.yml` | AI asset catalog |

These files are not included in the index files above. To load any of them, add an
individual `type: url` location entry pointing directly at the file.
