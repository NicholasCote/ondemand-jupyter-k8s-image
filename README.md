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
- `info.md.erb` prints the image on the session card, so several sessions
  running side by side can be told apart.

## Choosing an image

The image needs a `jupyter` executable and must be reachable from the Cirrus
cluster. `launch.sh` (generated into the pod's configmap by `submit.yml.erb`)
probes for both layouts in circulation rather than assuming one:

| Convention | Startup wrapper | Jupyter |
| --- | --- | --- |
| jupyter/docker-stacks | `/usr/local/bin/start.sh` | `/opt/conda/bin/jupyter` |
| repo2docker, cirrus | `/srv/start` | `/srv/conda/bin/jupyter` |

Anything else on `PATH` is used as a last resort. Examples:

- `quay.io/jupyter/scipy-notebook:latest`
- `quay.io/jupyter/datascience-notebook:latest`
- `quay.io/jupyter/tensorflow-notebook:latest`
- `hub.k8s.ucar.edu/cirrus-jhub/jhub-cpu-nb:0.1.5`

## Running as the real user

The pod's `securityContext` is the user's own UID and GID, which no stock image
knows about — its `/etc/passwd` was built around `jovyan` (UID 1000). Two things
follow, and `submit.yml.erb` handles both:

- **Identity.** `/etc/passwd` and `/etc/group` are supplied from the configmap
  with an entry for the real user, mounted with `subPath` so they replace those
  two files and nothing else in `/etc`. Without this, `getpwuid()` fails and the
  session comes up with an `I have no name!` prompt.

  **These files add to the image's accounts, they never replace them.** Keep
  `jovyan` (1000:100) and the `users` group in there. docker-stacks' `start.sh`
  runs under `set -e` and does `JOVYAN_UID="$(id -u jovyan 2>/dev/null)"` — drop
  the account and `id` exits 1, the assignment inherits that status, and the
  script dies before it ever execs Jupyter. The pod crashloops with exit 1 and
  no error message, and the last log line is the `start-notebook.d` hooks.
  Images using the `/srv/start` convention do not make that lookup, so a
  regression here looks like "works on our images, broken on Docker Stacks".
- **Home.** The container's `workingDir` is set to the user's NFS home. Images
  bake `WORKDIR /home/jovyan`, and a non-root `start.sh` never `cd`s to `$HOME`,
  so otherwise Jupyter's file browser and terminals open on a container-local
  directory that vanishes with the pod.

## Files

| File | Purpose |
| --- | --- |
| `manifest.yml` | App name, category, and description shown in OnDemand |
| `form.yml` | Launch form fields (image, CPUs, memory, wall time) |
| `submit.yml.erb` | Kubernetes container spec, mounts, configmap, init containers |
| `view.html.erb` | Session connect button |
| `info.md.erb` | Image shown on the session card |
| `template/` | `batch_connect` template directory |

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.
