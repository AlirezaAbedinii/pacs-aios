# Field notes

Thirty-eight problems that each cost real time while building PACS AIOS. They are
here because most of them share a shape: **something reported success while doing
nothing**. A ready node that could schedule nothing. A GPU that `nvidia-smi` shows and
CUDA cannot use. A chat page that returns HTTP 200 and an empty model list. The fix is
usually small. Finding it is not, so each note says what was seen, why it happened,
and where the fix lives.

File paths, patch numbers and the decision numbers in brackets refer to the prototype's
private repository, where each fix is recorded with the alternatives that were rejected.
How the pieces fit together is in [architecture.md](architecture.md).

---

## Bringing the stack up

### 1. `IS_PROD` must be set to `false` explicitly

PAPI's own config defaults it to `false`. Its Dockerfile then sets `ENV IS_PROD=True`.
Run the official image without overriding it and PAPI refuses to start, asking for
Harbor, Jenkins, provenance and LiteLLM tokens, in an error that does not point at the
cause. Leaving it unset is not enough. `compose/docker-compose.yml` sets it. [D-06]

### 2. A node that is "ready" can still schedule nothing

Every PAPI job template requires four things to line up: `meta.status=ready`,
`meta.type=compute`, a matching `meta.namespace`, and `region = "global"`. The stock
Ansible install ships `meta.status=test` and region `ai4os`, so it schedules nothing
while every node reports itself healthy. Only the cluster test suite flips the status.
`scripts/verify-cluster.sh` checks all four before anything else gets blamed.

### 3. Deploying the federated server calls Vault before Nomad

PAPI creates a secret and a Vault token before it schedules the federated-learning
server, against a Vault address hardcoded upstream. Without a Vault that trusts our
Keycloak, the headline demo failed inside PAPI with an error that mentioned neither.
Patched (`patches/ai4-papi/0002`), with a Vault bootstrapped to trust the realm. [D-14]

### 4. Instance names must not contain hyphens

This applies to the OpenStack instance name, so use underscores. Ansible then names the
Nomad and Consul agents `paios-wn-gpu-0` and so on, hyphens included. That is expected,
not a mistake.

### 5. Client nodes need an XFS volume at `/dev/vdb`

Docker's `storage-opt` disk limits only work on XFS, and the upstream cluster tests
assert that device path literally. Allow at least 5 GB per CPU core.

### 6. The scheduler must stay in `spread` mode

On `binpack`, two of the three "hospitals" land on the same machine and the demo quietly
stops being a three-site demo. A soft anti-affinity does not prevent it, so deployment
order matters too (see 28). [D-19]

---

## GPUs

### 7. PAPI caps a user at two GPUs

The cap counts across all of a user's running deployments, so three GPU-backed
federated clients under one account are rejected on the third. The clients run CPU-only.
The LLM tool escapes the cap entirely: its form has no `hardware` section, so the quota
check finds no GPU count and never counts it.

### 8. The GPU has 10.3 GB usable, not 12

The vGPU reports 12,288 MiB but only 10,565 MiB is free, because ECC and vGPU overhead
take 1,724 MiB. Anything that sizes itself against the nominal total overshoots and dies
at startup. It is a MIG-backed slice of an H100 with compute capability 9.0, so bfloat16
works and upstream's `--dtype float16` is unnecessary.

### 9. `nvidia-smi` is not evidence that a GPU works

These are MIG-backed vGPUs. Version 1.0.0 of Nomad's NVIDIA device plugin allocated the
**parent** device, which CUDA cannot address. Every job with a `device "gpu"` stanza,
which is every PAPI template, got a container where `nvidia-smi` showed the GPU and
`torch.cuda.is_available()` was `False`. Nothing GPU-backed computed anything for a
week, and every check looked fine.

Fixed by upgrading the plugin to 1.1.0 (`ansible/playbook-nvidia-plugin.yml`). The
regression test, `scripts/check-gpu-scheduling.sh`, multiplies two matrices on the
device, because nothing short of that proves anything.

### 10. The GPU's name changed with the fix

The 1.1.0 plugin reports the MIG instance, `NVIDIA H100L-1-12C MIG 1g.12gb`, with
10,564 MiB, where 1.0.0 reported `NVIDIA H100L-1-12C`. Everything that matches on the
model string had to follow: PAPI's GPU table, the workspace form's dropdown, and the
LLM tool's GPU allowlist.

### 11. Every module image predates the H100

`ai4os-yolo-torch` is PyTorch 1.13 on CUDA 11.6, which has no kernels for compute
capability 9.0, so the driver JIT-compiles them at the first CUDA call. It was still
going after 25 minutes on one core. Five of the eight modules are CUDA 9 or 10, which
cannot see a MIG slice at all. Modules run on CPU (YOLO takes 0.1 to 0.27 s an image),
and PAPI now refuses a module GPU with a 400. The workspaces' PyTorch 2 images and vLLM
have native Hopper kernels. `scripts/check-modules.sh` measured all eight. [D-81, D-83]

---

## Serving a language model

### 12. The LLM tool had four blockers on this hardware, not one

The "requires NVIDIA T4 GPUs" error was only the first. There was a second `Tesla T4`
constraint inside the Nomad job. The job asked for 8 CPU cores on 3-core nodes. And its
two helper tasks called the deployment's own public HTTPS address, so they died on the
self-signed CA. Because they ran as `prestart` and `poststart` hooks, that killed the
allocation. Patched (`patches/ai4-papi/0009`) plus our own job template.

### 13. Open WebUI fails silently when it cannot reach itself

The fourth blocker above was wider than the helper tasks. It was anything in a
deployment that reached its own services by public hostname, and the worst case was
Open WebUI itself. It catches the TLS error and serves `GET /api/models` as **200 with
an empty list**. The deployment is healthy, login works, and the model dropdown is
empty. Now every service in an allocation addresses its neighbours through Nomad's
own addresses (`patches/ai4-papi/0010`). If an LLM deployment looks fine and does
nothing, read the `open-webui` task's stderr: it is the only place that says why. [D-36]

### 14. The first visitor to a fresh Open WebUI becomes its administrator

Upstream claimed the admin account from outside, with a `poststart` HTTP signup. Between
the port opening and that POST landing there were 0 to 3 seconds, and a smoke test fell
into that window and took ownership of the deployment it was testing. Fixed with
`WEBUI_ADMIN_*`, which Open WebUI handles before uvicorn opens the socket. The signup
task is gone and must not come back: with an admin present, signup answers 403, which
upstream retries for fifteen minutes before failing the allocation. [D-37]

### 15. Ready needs three signals

The dashboard called a language-model deployment ready about three minutes before you
could chat with it. Each signal that looked right was wrong:

1. PAPI read the task it renames to `main`. For the LLM tool that is vLLM, a `prestart`
   sidecar, which went green about three minutes before Open WebUI existed.
2. Reading the right task still means reading a **container**. Uvicorn opens its socket
   only after the FastAPI lifespan has created the admin and closed signup (14).
3. Even Nomad's healthy verdict is early. Traefik's Consul catalogue provider polls
   every 15 seconds by default, and no health check can see that hop, so
   `min_healthy_time = "25s"` covers it.

The gap went 184 s → 22 s → 3 s → 0 (`patches/ai4-papi/0011`, health checks in
`configs/papi/tools/ai4os-llm/nomad.hcl`). Two facts cost time along the way:
`DeploymentStatus` is **absent** during the window, never `False`. And a group with no
checks falls back to task-state health, so the check is half the fix, not defence in
depth.

The same patch fixed two more lies. A deployment Nomad had queued for capacity showed as
a red `error` with an empty message, because `/evaluations` is unordered and `evals[0]`
was the blocked one. And every deployment flashed `error` for its first second.
[D-38, D-39]

### 16. While the federated demo is up, the cluster fits exactly one LLM

This is not a GPU limit: the three hospital nodes each have a free GPU. An LLM deployment
wants two exclusive CPU cores plus 1,300 MHz, and that does not fit in what is left of a
3-core node once a workspace holds a core. A second one queues, and now says so, naming
the resource that ran out.

### 17. The LLM catalogue page has three ways to look broken, all returning 200

The page reads one YAML file that PAPI also reads, so a change has to reach **both** or
the cards and the dropdown describe different platforms. A card's image is
`{family}_logo.png`, and a family with no badge renders broken, because nginx answers
every missing path with `index.html` and HTTP 200. And a model's id is the YAML key.
Upstream rebuilt it as `family + '/' + name`, which broke for `mistralai/…` and
`ibm-granite/…`. [D-40 to D-42]

---

## The dashboard and privacy

### 18. The dashboard talks to third-party hosts from the visitor's browser

There were four ways. Sidenav links, footer links **and an analytics beacon** sit in a
base config merged in at build time, and left alone the beacon reports every demo visit
to a third-party tracker. The tenant file overrides all three. The fourth was the LLM
catalogue, fetched from `raw.githubusercontent.com` until it was served from the
cluster (17).

### 19. "Platform status" notices came from another project's issue tracker

Upstream hardcodes AI4EOSC's GitHub issue tracker in three methods. Between them they
drive the startup popup, the notifications bell and the red maintenance banner, all
fetched from the visitor's browser, twice per page load. GitHub allows 60
unauthenticated requests an hour per IP, so about thirty page loads from one office and
every visitor sees an error instead. And a notice with no VO displays everywhere, so
another platform's maintenance window rendered as our banner. The source is now a tenant
setting, and blank means no request at all (`patches/ai4-dashboard/0005`). [D-62]

### 20. Never rename a module's `categories`

The AI4OS metadata schema enumerates the categories, PAPI validates every entry, and one
renamed value turns an entry into "invalid metadata" with its id for a title. All eight
modules went that way, live, for five minutes. The dashboard also tells a tool from a
module by the value `AI4 tools`. Relabel at display time instead
(`patches/ai4-dashboard/0017`). The catalogue mirror may only drop VO tags and fix
titles. [D-84]

### 21. Two faults only a browser could see

Upstream's `formatDate` read PAPI's zone-less UTC timestamps as browser-local, then
formatted them in Europe/Paris (`patches/ai4-dashboard/0016`). A dimmed card's tooltip
could never open, because it sat under `pointer-events: none`
(`patches/ai4-dashboard/0018`). Every script passed both. A walk through the pages in a
browser found them.

---

## Identity and access

### 22. "Pending" is not stored anywhere

A pending user *is* a realm user with no `access:<vo>:<level>` role. Approval is one role
assignment, denial is one disable, and there is no database to keep in step with
Keycloak, so approving in the Keycloak console works exactly as well as the admin page.
Three consequences:

- A user with no access role made PAPI answer HTTP 500, because it passed `None` to
  `list.index()`. That never fired until self-registration existed
  (`patches/ai4-papi/0018`).
- `registrationAllowed` is a **live** realm setting, and the realm import runs only
  once, so the bootstrap script re-applies it.
- The approval service holds `view-users` and `manage-users`, and must never be given
  `view-realm` or `realm-admin`. It reads role representations through the
  available-role-mappings endpoint instead.

[D-70 to D-72]

### 23. OSCAR authorises on a group, not a role

OSCAR chooses the claim it reads by a substring match on the token issuer. Our realm
matches neither of its special cases, so it reads `group_membership`, and the access
role every token carries is invisible to it. The group, its mapper and the demo users
were first set up by hand, so every account created afterwards was refused with 401.
Now the bootstrap script creates them, and the approval service grants the group on
approve. [D-74]

### 24. An OSCAR error arrived as a blank 500, with the cause destroyed

`detail=<exception object>` is not JSON, so FastAPI raised `TypeError` inside its own
error handler and returned an empty 500. The inference page polls every 5 seconds, so
one unauthorised account produced an "Error calling the API" banner every 5 seconds
that said nothing. Now the error is `str(e)`, sent as a **502**, which the dashboard
shows as a notice instead of ejecting the user to `/forbidden` (`patches/ai4-papi/0019`).
[D-73]

---

## Networking and TLS

### 25. NVIDIA FLARE needs TCP 8002 and 8003

The Traefik job already defines the matching entrypoints, so this is purely a
security-group item (`scripts/openstack-security-groups.sh`).

### 26. A machine nothing in the repository configured

The public address is a separate VM running nginx. It redirects `:80` to `:443`,
terminates TLS, and routes by Host to Caddy or Traefik. Meanwhile `/etc/hosts` on the
control plane mapped the public names straight to Caddy, so every `curl` from that node
bypassed the proxy and measured a different path from the one visitors take. The
documentation said for a week that no such machine existed. To measure what a visitor
gets, pin the name to the proxy with `--resolve` and read the `Server:` header.
[D-69]

### 27. The scheme is one variable

Caddy's site blocks, Keycloak's issuer, PAPI's advertised endpoints, the dashboard's API
URL, sixteen Traefik router tags, the federated bundles and every check script derive
from `PAIOS_SCHEME`. Rollback is the variable, a render and a restart, with no rebuild.
Both schemes always answer, and the inactive one redirects to the active one, because
browsers upgrade `http://` navigations on their own and would otherwise strand a visitor
on a page whose API calls are blocked as mixed content. One router keeps TLS whatever
the setting: the federated server's gRPC on :443. [D-67, D-68]

### 28. The proxy cannot carry gRPC

A deployment's public name leads to the proxy even from inside the cluster. The
federated clients followed DNS there, and every stream was reset (`RST_STREAM`, error
code 1) while the server waited for ever. The documentation claimed the clients bypassed
the proxy, and nobody had measured it. Now the hospital bootstrap pins the server's name
to the ingress node in `/etc/hosts`, and `scripts/check-fl-cluster.sh` checks the whole
path. Also: deploy the federation before any other workspace, or spread scheduling may
put two hospitals on one machine. [D-87]

---

## Images and storage

### 29. Platform storage is Nextcloud over WebDAV, not S3

The rclone vendor list offers only `nextcloud`, and the storage code hardcodes the
Nextcloud DAV path. MinIO appears upstream only under OSCAR.

### 30. CVAT is 22 containers in one Nomad group

That is about 71 GB of RAM, all on one node, and no node in this layout can host it.
Deferred until a larger node exists.

### 31. The image garbage collector deletes your pre-pulled images

`docuum` runs as a Nomad system job on every compute node and evicts least-recently-used
images above a threshold upstream hardcodes at 50 GB. The full pre-pull set is 67.9 GB,
and `vllm/vllm-openai` alone is 30.8 GB on disk (10.5 GB is the compressed registry
figure). So the pre-pull playbook reported success for every image while docuum deleted
half of them. Ours runs at 80 GB (`nomad-jobs/docuum.hcl`), and because re-running the
Nomad playbook reverts it, `scripts/verify-cluster.sh` checks the live value.

### 32. Nomad re-pulls a `latest` tag at every deployment, whatever `force_pull` says

Nomad's Docker driver looks for a local copy only for a pinned tag or a digest. So
pre-pulling protects an image only if the job names it the same pinned way. The module
template's UI sidecar pulled `deepaas_ui:latest` from an external registry. One day
that registry answered its health check and hung on every manifest, and a module died
with the image already sitting on the node. The image is now pinned by digest
(`patches/ai4-papi/0020`), and `tests/test_module_template.py` keeps the template and
the pre-pull list equal. [D-80]

### 33. OSCAR answers with whatever the script prints

A synchronous OSCAR call returns the script's output, stderr included, unless it finds a
file in the output folder, which it returns base64-encoded. Serverless calls now answer
with their result as plain JSON (`patches/ai4-papi/0022`). A service keeps the script it
was created with, so a service older than the patch still answers with its log until it
is recreated.

### 34. Pre-pulling does not keep an image; a referencing container does

docuum evicts the least recently used image first, and an image pulled weeks ago goes
first. After a sweep of module checks, the LLM node had no vLLM image at all. docuum never
deletes an image that a container references, so the pre-pull playbook now gives each demo
image a never-started pin container, per node role (`ansible/files/paios-pin.sh`).
[D-86]

---

## Scripts and builds

### 35. Never write `cmd | grep -q` under `set -o pipefail`

It reports failure **when the pattern is found**. grep exits at the first match, the
writer takes SIGPIPE, and pipefail returns the writer's status. This cost two debugging
rounds in one day. Capture the output first, then match a here-string.
`tests/test_oscar_authorisation.py` enforces it. [D-75]

### 36. Docker creates an empty directory for a missing bind mount

Docker needs `sudo` here, and `sudo` resets `HOME` to `/root`. A Compose file that
mounted from `${HOME}` therefore pointed at a path that did not exist, and Docker
silently created an empty **directory** there instead of failing. That is how PAPI came
to run healthily while trusting no certificate of ours. Nothing in `compose/` mounts
from `${HOME}`. [D-29]

### 37. Replacing a bind-mounted directory serves a deleted one

Caddy serves the federated bundles from a bind-mounted directory. Recreating that
directory gives it a new inode, and Caddy keeps serving the deleted one, so every bundle
URL returns 404. `scripts/build-fl-bundles.sh` empties the directory in place. [D-44]

### 38. Workspaces verify TLS; they never turn it off

The workspaces reach platform endpoints with the CA from their own bundle
(`SSL_CERT_FILE`), never `verify=False`. The one unverified request in the whole demo is
the `curl -k` that fetches the bundle that carries the CA. [D-43]

