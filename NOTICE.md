# Attribution Notice

## Upstream

This repository is a derivative work of **Taskosaur**, published by
**NetTantra Technologies (India) Private Limited** and licensed under the
**Business Source License 1.1**.

- Licensor: NetTantra Technologies (India) Private Limited
- Licensed Work: Taskosaur, © 2025 NetTantra Technologies (India) Private Limited
- Change Date: four years from the publication date of the Licensed Work
- Change License: MPL 2.0
- Licensing contact: licensing@nettantra.com

The full license text is retained verbatim in [LICENSE.md](LICENSE.md).
BSL 1.1 §"You must conspicuously display this License on each original or
modified copy of the Licensed Work": that requirement is why this file and
`LICENSE.md` exist, and it survives rebranding.

## What changed in this derivative

This copy has been adapted for internal deployment as the Enterprise Project
Management Suite (EPMS). The modifications are limited to packaging,
infrastructure, and documentation:

| Area | Change |
|---|---|
| Branding | `Taskosaur` → `EPMS` across source, configuration, and brand assets. Upstream trademarks and logos removed; the BSL grants no trademark rights. |
| Packaging | npm scope `@taskosaur/*` → `@epms/*`; version set to `1.0.0-enterprise`; maintainer metadata updated. |
| Registry | Public Docker Hub coordinates replaced with `registry.internal.company.com/pm-app`, overridable via `INTERNAL_REGISTRY` / `IMAGE_NAME`. |
| Secrets | Live JWT and encryption keys removed from the committed `.env.example`; replaced with placeholders. |
| CI | Added `.github/workflows/enterprise-build.yml`: isolated per-suite integration runs over `docker-compose.dev.yml`. |
| Commit gate | `.husky/pre-commit` extended to enforce Prettier in addition to ESLint. |
| Docs | `README.md` and `DOCKER_DEV_SETUP.md` rewritten as internal engineering documentation. |
| Removed | `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `SECURITY.md`: upstream community process documents, not applicable to an internal fork. |
| Removed | `docker/README.md`: upstream's public Docker Hub listing page. Obsolete for a private internal registry; its badges and community links no longer resolved. |

Application logic, data model, and domain modules are upstream work and are
**not** original to this repository.

## Obligations

Anyone using, modifying, or redistributing this code remains bound by BSL 1.1:

- Retain `LICENSE.md` and this notice on every copy, original or modified.
- Internal organisational use (including self-hosting) is permitted under the
  Additional Use Grant.
- Offering this software to third parties on a hosted or embedded basis as a
  competing paid product is **not** permitted without a commercial license from
  the licensor.
- Non-compliant use automatically terminates the license grant.

If the intended use may fall outside the Additional Use Grant, obtain written
confirmation from the licensor before proceeding.
