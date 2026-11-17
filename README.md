# Jupyter (CIRRUS)

An OpenOnDemand app that launches a Jupyter session in a container on the
**Cirrus** Kubernetes cluster. You choose the container image on the launch
form, and your HPC home directory is mounted into it — the same files and
notebooks you see from a login node. An NVIDIA GPU can be requested.

The interface is JupyterLab. Terminals, `conda activate`, and `nvidia-smi` on a
GPU session all work as they do elsewhere on the system.

## The launch form

- **Container image** — any Jupyter image reachable from Cirrus. Both common
  layouts are supported (`jupyter/docker-stacks` and repo2docker-style images);
  the session finds whichever the image uses. Some that work:

  - `hub.k8s.ucar.edu/cirrus-jhub/jhub-cpu-nb:0.1.5` (the default)
  - `quay.io/jupyter/scipy-notebook:latest`
  - `quay.io/jupyter/datascience-notebook:latest`
  - `quay.io/jupyter/tensorflow-notebook:latest`

- **CPUs** — 1 to 8.
- **Memory** — 1 to 32 GB.
- **GPUs** — check the box for one NVIDIA GPU. The image has to be able to use
  it: `pytorch-notebook` and `tensorflow-notebook` can, plain `scipy-notebook`
  cannot.

## Session limits

- **About 1 hour with no browser connected.** The session shuts itself down,
  so a forgotten notebook does not hold a GPU or a node. Closing the tab,
  sleeping the laptop, or losing the network all start that clock. A notebook
  that is still running a cell keeps the session alive, and so does a tab left
  open, even if nobody is using it.
- **Terminals close after 1 hour without input or output**, even with the tab
  open. A long command that prints nothing for an hour is ended with its
  terminal. Run it from a notebook, or under `nohup` with output to a file.

There is no enforced maximum session length on Cirrus today. `submit.yml.erb`
sets a 12-hour lifetime, but that is only an annotation, and the
[job-pod-reaper](https://github.com/OSC/job-pod-reaper) that would act on it is
not running there. The idle shutdown above is what ends sessions.

An ended session stays on the dashboard marked **Completed** rather than
disappearing, and you can start another whenever you like. Each launch is a
fresh container, though, so keep your work in your home directory — anything
written elsewhere does not carry over.

## Connecting

Launch the app, then use **Connect to Jupyter** on the session card. Each
session gets its own password and the button submits it for you; there is no
token to copy from a log.

## Files

| File | Purpose |
| --- | --- |
| `manifest.yml` | App name, category, and description shown in OnDemand |
| `form.yml` | Launch form fields |
| `submit.yml.erb` | Pod spec, mounts, configmap, init containers |
| `view.html.erb` | Session connect button |
| `info.md.erb` | Image and GPU count on the session card |
| `template/` | `batch_connect` template directory |

`submit.yml.erb` is commented in detail: how the session runs as your real UID
in an image built for another one, where the idle and wall-time limits above are
set, how `conda activate` is made to work in terminals, and how the session
password reaches Jupyter. Read those comments before changing it — most of what
looks redundant there is load-bearing.

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.
