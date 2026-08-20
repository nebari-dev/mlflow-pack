---
title: Introduction
description: Documentation for the Nebari MLflow Pack - a Helm chart that deploys MLflow on Nebari with Keycloak authentication, PostgreSQL, and TLS.
---

The Nebari MLflow Pack deploys [MLflow](https://mlflow.org/) on
[Nebari](https://nebari.dev) with Keycloak authentication, a PostgreSQL backend
store, and automatic TLS.

:::note[Documentation in progress]
This site is the scaffolding for the pack's documentation. Content is being
written; until it lands here, the
[repository README](https://github.com/nebari-dev/mlflow-pack#readme)
is the reference for installing and configuring the pack.
:::

## Reference

- [Authentication flow](/auth-flow/) — how OIDC authentication works for Nebari
  software packs.
- [NebariApp CRD reference](/nebariapp-crd-reference/) — field-by-field
  reference for the NebariApp custom resource.

## Contributing to these docs

Pages live in `docs/src/content/docs/`. See the
[docs README](https://github.com/nebari-dev/mlflow-pack/blob/main/docs/README.md)
for how to run the site locally and add a page.
