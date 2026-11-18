# Deployment for Renovate

Repository containing manifests for [Renovate](https://github.com/renovatebot/renovate).

This repository uses the Helm hydration workflow.

## Dependencies

This chart references `renovate` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Workshop

The hands-on exercises are in [WORKSHOP.md](./WORKSHOP.md).

## Slides

The workshop slides are in [slides/workshop.pdf](./slides/workshop.pdf).
They are only a fallback for the presenter, the labs are in [WORKSHOP.md](./WORKSHOP.md).

## License

Copyright 2026 Steadforce GmbH. Licensed under the [Apache License, Version 2.0](./LICENSE).

The slides in [slides/](./slides/), including the photos and the Steadforce branding, are
not covered by this license. All rights reserved.

The workshop uses third-party software under its own licenses, for example the
[Renovate](https://github.com/renovatebot/helm-charts) Helm chart (AGPL-3.0) and
[Argo CD](https://github.com/argoproj/argo-cd) (Apache-2.0).
