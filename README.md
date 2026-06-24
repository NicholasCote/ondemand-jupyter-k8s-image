# Jupyter (Kubernetes, custom image) — OnDemand app

An OpenOnDemand `batch_connect` app that launches Jupyter in the **Cirrus**
Kubernetes cluster using a **container image the user specifies** on the launch form.

This is **Phase B** of a three-part effort:

- **Phase A:** fixed image — [`ondemand-jupyter-k8s`](https://github.com/NicholasCote/ondemand-jupyter-k8s).
- **Phase B (this repo):** user supplies any image.
- **Phase C:** Binder-style build from a git URL — [`ondemand-jupyter-binder`](https://github.com/NicholasCote/ondemand-jupyter-binder).

## What's different from Phase A

- `form.yml` adds a required **Container image** text field (`custom_image`).
- `submit.yml.erb` uses `<%= custom_image.strip %>` as the container image.

The image must be a Jupyter Docker Stacks-compatible image (contains
`/usr/local/bin/start.sh` and `jupyter`) and reachable from Cirrus. Examples:
`quay.io/jupyter/scipy-notebook:latest`, `quay.io/jupyter/datascience-notebook:latest`.

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this repo's
**HTTPS** git URL. App files are at the repo root so OnDemand finds `manifest.yml`.

See the Phase A repo for the first-launch verification checklist (helper image,
`$HOME` mount, RBAC, Jupyter config class) — it applies here too.
