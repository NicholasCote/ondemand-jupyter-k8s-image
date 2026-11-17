# Jupyter (CIRRUS)

An OpenOnDemand app that starts JupyterLab in a container on the **Cirrus**
Kubernetes cluster. You pick the container image on the launch form, and your
GLADE storage is mounted into it, so you see the same files as on a login node:

- your home directory
- `/glade/work`
- `/glade/campaign` (read-only)
- `/glade/collections` and `/gdex`

Terminals, `conda activate`, and `nvidia-smi` on a GPU session work as they do
elsewhere on the system.

## The launch form

- **Container image**: any Jupyter image Cirrus can pull. The default is
  `quay.io/jupyter/scipy-notebook:2026-11-16`. Other
  [Jupyter Docker Stacks](https://quay.io/organization/jupyter) images work
  too, such as `datascience-notebook`, `pytorch-notebook`, or
  `tensorflow-notebook`. Give a tag so the image doesn't change under you.
- **CPUs**: 1 to 8.
- **Memory**: 1 to 32 GB.
- **GPUs**: check the box for one NVIDIA GPU. The image has to be able to use
  it: `pytorch-notebook` and `tensorflow-notebook` can, plain `scipy-notebook`
  cannot.

## Connecting

Launch the app, then click **Connect to Jupyter** on the session card. Each
session gets its own password, and the button logs you in.

## When a session ends

- **After 1 hour with no activity.** Activity means running a notebook cell,
  typing in or getting output from a terminal, or anything still computing,
  including a long cell or a background job. An open browser tab on its own
  doesn't count, and neither does just reading or editing files.
- **After 12 hours**, whatever it is doing.

An ended session shows as **Completed** on the dashboard, and you can launch a
new one whenever you like. Each launch is a fresh container, so keep your work
in your home directory or on GLADE; anything saved elsewhere is gone when the
session ends.

## Files

| File | Purpose |
| --- | --- |
| `manifest.yml` | App name, category, and description shown in OnDemand |
| `form.yml` | Launch form fields |
| `submit.yml.erb` | Pod spec, mounts, and the `launch.sh` that starts Jupyter and ends idle sessions |
| `view.html.erb` | Session connect button |
| `info.md.erb` | Image and GPU count on the session card |
| `template/` | `batch_connect` template directory |

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.
