# Jupyter (Kubernetes, custom image) — OnDemand app

An OpenOnDemand `batch_connect` app that launches Jupyter Lab in the **Cirrus**
Kubernetes cluster using a **container image the user specifies** on the launch
form.

## What it does

- The launch form asks for a container image plus CPUs, memory, and wall time.
- `submit.yml.erb` renders a Kubernetes pod spec that runs Jupyter Lab from that
  image on port 8080, with the user's NFS home directory mounted at `$HOME`.
- Init containers (from `ohiosupercomputer/ood-k8s-utils`) generate a per-session
  password, hash it into the Jupyter config, and set `base_url` to the assigned
  node and NodePort so OnDemand can proxy the session.
- `view.html.erb` renders a **Connect to Jupyter** button that posts the
  generated password to log the user in.

## Choosing an image

The image must be Jupyter Docker Stacks-compatible — it needs
`/usr/local/bin/start.sh` and `/opt/conda/bin/jupyter` — and must be reachable
from the Cirrus cluster. Examples:

- `hub.k8s.ucar.edu/cirrus-jhub/jhub-cpu-nb:0.1.5` (form default)
- `quay.io/jupyter/scipy-notebook:latest`
- `quay.io/jupyter/datascience-notebook:latest`
- `quay.io/jupyter/tensorflow-notebook:latest`

## Files

| File | Purpose |
| --- | --- |
| `manifest.yml` | App name, category, and description shown in OnDemand |
| `form.yml` | Launch form fields (image, CPUs, memory, wall time) |
| `submit.yml.erb` | Kubernetes container spec, mounts, configmap, init containers |
| `view.html.erb` | Session connect button |
| `template/` | `batch_connect` template directory |

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.
