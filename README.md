# Dune Docker Addons

Community addon index for Dune Docker Console.

This repository is the reviewed addon list used by Dune Docker Console. It does not host addon source code. Addon source code should live in each developer's own addon repository, while this repo points to reviewed release manifests and packages.

## Addon Authors

Start from the addon template repo:

```text
https://github.com/Red-Blink/dune-docker-addon-template
```

Build and release your addon from your own repository. To list it here, add a manifest under `addons/` and include it in `index.json`.

See `docs/addon-submission.md` and `docs/addon-manifest.md` for the expected format.

See `docs/addon-upstream-pr-workflow.md` for a reproduction guide to the
CI workflows, validation scripts, security gates, packaging, and
catalog-PR automation used by an existing listed addon
(`dune-ops-observability`), if you want to script the same pipeline for
your own addon repository.

## Addon Lifecycle

The community index can mark addons as `active`, `deprecated`, `unsupported`, `removed`, or `blocked`.

Lifecycle state is managed in this repository, not inside an addon's own `addon.json`. This lets the catalog stop recommending abandoned addons, warn users about unsupported addons, and block unsafe addons while still letting server owners uninstall local copies cleanly.
