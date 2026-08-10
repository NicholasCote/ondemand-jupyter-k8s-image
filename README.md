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

  **Keep these files minimal and never pad them with system accounts.** Distro
  families disagree on the numbering — Debian has `daemon`=1 / `bin`=2, RHEL has
  `bin`=1 / `daemon`=2 — so entries copied from one are wrong on the other, and
  a wrong name is worse than a missing one because `ls -l` repeats it.

  `/etc/group` is therefore **seeded from the OnDemand host's own group
  database** rather than from a hardcoded list. `submit.yml.erb` copies every
  local group with GID < 1000 out of the host's `/etc/group` verbatim.

  This exists so nothing in a fresh session prints a warning. Mounting the file
  hides the names the image shipped, and a GID with no name makes `groups` emit

  ```
  groups: cannot find name for group ID 39
  ```

  which an end user cannot distinguish from a real fault. GID 39 shows up on
  GPU sessions: the runtime attaches the group that owns `/dev/nvidia*` so the
  container can open the device. It is what lets `nvidia-smi` see the card.

  Taking the names from this host is not a workaround, it is the correct
  source. Supplementary GIDs are assigned by the **node**, not by the image, so
  the node's numbering is the one that has to be honoured, and the OnDemand
  host shares it. Reading `/etc/group` directly rather than using `Etc.group`
  is deliberate: `Etc.group` walks all of NSS including LDAP, and only local
  system groups belong in a container. The `< 1000` cut keeps it to the system
  range where injected device groups live.

  If a GID still comes up unnamed, check that this host knows it —
  `getent group <gid>` on the OnDemand host — before changing anything here.
- **Home.** The container's `workingDir` is set to the user's NFS home. Images
  bake `WORKDIR /home/jovyan`, and a non-root `start.sh` never `cd`s to `$HOME`,
  so otherwise Jupyter's file browser and terminals open on a container-local
  directory that vanishes with the pod.

## Idle sessions

Sessions end themselves after **1 hour idle**, configured in
`submit.yml.erb` (`idle_timeout`, one place to change it). Idle kernels and
terminals are culled first, because the server only considers shutting down
once nothing is running; then it exits 0, and `restart_policy: OnFailure`
leaves the pod `Completed` rather than restarting it, releasing the GPU.

This needs no cluster support, which is why it is done here rather than with
`wall_time`. Kubernetes has no idea how long a Pod has been idle, or how long
it has been running at all — see the note on wall time below.

Verified end to end: an abandoned kernel is culled, then the server exits.

```
Culling kernels with idle durations > N seconds at M second intervals
Culling 'python3' (5f395a0b…) with 0 connections due to inactivity
-> server exits, code 0
```

**What this does and does not catch.** `shutdown_no_activity_timeout` measures
time since the last **HTTP request**, not since the last human action. Measured
directly: a client polling the API held the server open indefinitely, and it
shut down 40s after polling stopped. So

- browser closed, laptop asleep, network gone → reclaimed. This is the case
  that strands GPUs, and it is covered.
- a tab left open and in the foreground → JupyterLab keeps polling, and the
  session stays up no matter how long nobody touches it.

`cull_connected` stays `False` on purpose. Culling a kernel that still has a
browser attached destroys the user's variables while the tab keeps polling, so
the pod stays up and the GPU is not returned — all of the cost, none of the
benefit. Only `wall_time` plus a reaper closes that last gap.

## GPUs

The form's **GPUs** field sets `gpus_per_node`, which ood_core renders into the
pod's resource limits *and* requests as `nvidia.com/gpu: <n>`. The key comes
from `native.gpu_type`, which defaults to `nvidia.com/gpu`; set it in
`submit.yml.erb` if the cluster advertises GPUs under a different name.

Leaving the field at 0 omits the line entirely rather than emitting
`nvidia.com/gpu: 0`, so a CPU-only session's pod spec is unchanged. ood_core
guards the field with `unless script.gpus_per_node.nil?` — a nil check, not a
zero check — so the ERB has to skip the key, not zero it.

Requesting a GPU does not make an image able to use one. The image needs CUDA
and GPU-enabled framework builds; `quay.io/jupyter/pytorch-notebook` and
`tensorflow-notebook` qualify, the plain `scipy-notebook` does not.

### If a GPU session sits in Pending

**ood_core's pod template emits `nodeSelector` but has no `tolerations` support
at all.** If the GPU nodes carry a taint — `nvidia.com/gpu=present:NoSchedule`
is a common one — this app cannot schedule onto them, and no change to
`submit.yml.erb` will fix that, because there is no field to set. Check with:

```bash
kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu,TAINTS:.spec.taints'
```

If the GPU nodes are tainted, the options are to remove the taint, admit these
pods with a mutating webhook, or carry a patched pod template. If they are
merely labelled, add the label under `native.node_selector` in
`submit.yml.erb`:

```yaml
    node_selector:
      nvidia.com/gpu.present: "true"
```

## `conda activate` in session terminals

Sessions mount a small `/etc/profile.d/zz-ood-conda.sh` that evaluates the
image's conda shell hook. Without it, a terminal gives

```
CondaError: Run 'conda init' before 'conda activate'
```

and running `conda init` does not help — it reports success, and the next
terminal fails the same way. Two things combine:

- `jupyter_server_terminals` appends `-l` to the shell whenever the server's
  stdout is not a tty, which in a pod it never is. So terminals are **login
  shells**, and a login shell reads `/etc/profile` and `~/.bash_profile` —
  never `~/.bashrc`, which is the only file `conda init` writes to.
- Docker Stacks images keep their conda hook in `/home/jovyan/.bashrc` and ship
  no `/etc/profile.d` entry. Since `$HOME` is the user's NFS home, that file is
  not read either.

`/etc/profile.d` is the one hook both distro families source for login shells.
Mounting there is additive — a filename no image ships, in a directory that
already exists — so unlike `/etc/passwd` it hides nothing. The conda prefix is
probed (`/opt/conda`, `/srv/conda`, then `PATH`) rather than assumed, since all
three are in use across the images launched here.

## Session password

The `add-passwd-to-cfg` init container writes the generated password into the
config under three trait names, because the right one depends on how old the
image's Jupyter is:

| Trait | Applies to |
| --- | --- |
| `c.PasswordIdentityProvider.hashed_password` | `jupyter_server` >= 2.0 |
| `c.ServerApp.password` | `jupyter_server` 1.x |
| `c.NotebookApp.password` | classic `notebook` |

Writing only the first is a silent failure on an older image. Traitlets warns
about an unknown *option* on a class it knows, but ignores an entire unknown
class section without comment, so the server keeps token authentication, the
Connect button's POST gets a 401, and the user lands on **"Token authentication
is enabled"** with nothing in the log to explain it. If that page comes back,
check the image's `jupyter_server` version before anything else.

Note that `ood-k8s-utils`' `create_salt_and_sha1` does `SALT="$(create_passwd 16)"`,
but `create_passwd` exports `PASSWORD` rather than writing to stdout — so the
salt is always empty and hashes are unsalted `sha1(password)`. It authenticates
correctly; it is just weaker than it looks. That is upstream, not this app.

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
