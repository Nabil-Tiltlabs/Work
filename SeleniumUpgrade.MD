# Selenium Grid Upgrade & Deployment Guide

**Purpose:** Step-by-step procedure for bumping Selenium Grid image versions (hub + Chrome node), validating locally, and deploying/verifying in a Kubernetes (GKE) environment. Based on the 4.20.0 → 4.48.0 upgrade (Issue #2202 — CVE remediation + client/Grid version skew fix).

---

## 0. Background / Why

- The AVC crawler DAGs (Airflow/Composer) need a Selenium Grid (hub + Chrome nodes) to drive browser automation.
- The Grid images can go stale and accumulate CVEs (esp. in the bundled JRE).
- The Python `selenium` client version should match the Grid's version — a large version gap ("skew") is a latent source of failures (e.g. account-switcher / element-lookup issues).
- Files that reference the Grid images:
  - `docker-compose.yml` (local dev)
  - `kubernetes_deployments/selenium-hub-deployment.yaml`
  - `kubernetes_deployments/selenium-node-chrome-deployment.yaml`
  - `kubernetes_deployments/selenium-hub-vpc-deployment.yaml` (VerticalPodAutoscaler — no image field, no change needed)
  - `kubernetes_deployments/selenium-node-chrome-vpc-deployment.yaml` (VerticalPodAutoscaler — no image field, no change needed)
  - `kubernetes_deployments/selenium-hub-svc.yaml` (Service — no image field, no change needed)

---

## Phase 1 — Establish a Baseline on the OLD Version (Local, Docker Compose)

**Do this BEFORE changing any image tags**, so you have something to compare against.

1. Validate the Compose file:
   ```
   docker compose config --services
   ```
   Expect to see `selenium-hub`, `chrome`, plus other services.

2. Check if old images already exist locally:
   ```
   docker images | grep selenium
   ```

3. Start only the Selenium services (not the whole stack):
   ```
   docker compose up -d selenium-hub chrome
   docker compose ps
   ```
   Expect both `Up`.

4. Check logs for clean startup and node registration:
   ```
   docker compose logs selenium-hub --tail=100
   docker compose logs chrome --tail=100
   ```

5. Check Grid status:
   ```
   curl http://localhost:4444/status
   ```
   Look for `"ready": true`.

6. Open `http://localhost:4444` in a browser — confirm the Grid UI shows the Chrome node.

7. Run a minimal functional test. Create `test_grid.py`:
   ```python
   from selenium import webdriver
   from selenium.webdriver.chrome.options import Options

   options = Options()

   driver = webdriver.Remote(
       command_executor="http://localhost:4444/wd/hub",
       options=options,
   )

   try:
       driver.get("https://example.com")
       print("Page title:", driver.title)
       print("Current URL:", driver.current_url)
   finally:
       driver.quit()
   ```
   Run it:
   ```
   python3 test_grid.py
   ```
   Expect:
   ```
   Page title: Example Domain
   Current URL: https://example.com/
   ```

8. **Record the baseline** (paste into your PR/notes later):
   ```
   Hub: selenium/hub:<OLD_VERSION>
   Node: selenium/node-chrome:<OLD_VERSION>
   Python Selenium: <CLIENT_VERSION>

   Hub starts:        PASS
   Node registers:     PASS
   Grid /status ready: PASS
   test_grid.py:       PASS
   ```

---

## Phase 2 — Bump the Image Tags (Local Files)

Change **only** the `image:` lines — nothing else (env vars, probes, resources stay untouched unless there's a documented reason to change them).

**`docker-compose.yml`:**
```diff
  selenium-hub:
-   image: selenium/hub:<OLD_VERSION>
+   image: selenium/hub:<NEW_VERSION>

  chrome:
-   image: selenium/node-chrome:<OLD_VERSION>
+   image: selenium/node-chrome:<NEW_VERSION>
```

**`kubernetes_deployments/selenium-hub-deployment.yaml`:**
```diff
-   image: selenium/hub:<OLD_VERSION>
+   image: selenium/hub:<NEW_VERSION>
```

**`kubernetes_deployments/selenium-node-chrome-deployment.yaml`:**
```diff
-   image: selenium/node-chrome:<OLD_VERSION>
+   image: selenium/node-chrome:<NEW_VERSION>
```

> Use a date-stamped node-chrome tag (not a floating tag) for reproducibility, consistent with existing convention.

**Sanity check the diff before committing:**
```
git status
git diff
```
Confirm only the image tag lines changed.

---

## Phase 3 — Re-validate Locally (Docker Compose, NEW Version)

Repeat Phase 1's steps 3–7 against the new images:

```
docker compose down selenium-hub chrome
docker compose up -d selenium-hub chrome
docker compose ps
docker compose logs selenium-hub --tail=100
docker compose logs chrome --tail=100
curl http://localhost:4444/status
curl -i http://localhost:4444/wd/hub/status     # <-- the ACTUAL path k8s probes use, not just /status
python3 test_grid.py
```

Both `/status` and `/wd/hub/status` should return `200` with `"ready": true`. Compare all results side-by-side with the Phase 1 baseline.

---

## Phase 4 — Deploy to a Kubernetes Cluster (e.g. GKE dev environment)

### 4a. One-time tooling setup (only needed once per machine)

1. Install `kubectl`:
   ```
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

2. Install/configure `gcloud`, set the project, and authenticate to the cluster:
   ```
   gcloud config get-value project
   gcloud config set project <project_id>
   gcloud container clusters get-credentials <cluster_name> --region <region>
   ```
   Example used in this run:
   ```
   gcloud container clusters get-credentials us-central1-dev-composer-en-17738014-gke --region us-central1 --project mercury-dev-environment
   ```

### 4b. Check current state BEFORE applying anything

```
kubectl get deployment selenium-hub -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
kubectl get deployment selenium-node-chrome -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
kubectl get pods -n composer-user-workloads
kubectl get svc -n composer-user-workloads
```

Also check for any duplicate/stray Selenium deployments in other namespaces (e.g. `default`):
```
kubectl get deployment -n default | grep selenium
kubectl get svc -n default | grep selenium
```
If duplicates exist and are not needed, remove them:
```
kubectl delete -f <file>.yaml -n default
```

### 4c. Apply the updated manifests

```
kubectl apply -f kubernetes_deployments/selenium-hub-deployment.yaml -n composer-user-workloads
kubectl apply -f kubernetes_deployments/selenium-node-chrome-deployment.yaml -n composer-user-workloads
kubectl apply -f kubernetes_deployments/selenium-hub-svc.yaml -n composer-user-workloads
```

> Note: `selenium-hub-vpc-deployment.yaml` / `selenium-node-chrome-vpc-deployment.yaml` are VerticalPodAutoscalers — no image field, so applying them is optional/unaffected by the version bump. Apply them if they haven't been applied before, otherwise skip.

### 4d. Watch the rollout — don't just check once

```
kubectl rollout status deployment/selenium-hub -n composer-user-workloads
kubectl rollout status deployment/selenium-node-chrome -n composer-user-workloads
```
Wait for `successfully rolled out` on both. For single-replica deployments (hub), expect a brief window of unavailability during the swap — this is normal.

### 4e. Confirm final state

```
kubectl get pods -n composer-user-workloads
kubectl get deployment selenium-hub -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
kubectl get deployment selenium-node-chrome -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
```
All pods should be `Running`, `0` restarts, and images should match the new version tags.

**If pods are missing / stuck / crashlooping**, diagnose with:
```
kubectl describe deployment <name> -n composer-user-workloads
kubectl get rs -n composer-user-workloads | grep selenium
kubectl logs -n composer-user-workloads <pod-name>
kubectl describe pod -n composer-user-workloads <pod-name>
```
Check the `Events:` section at the bottom of `describe` output — it usually names the exact failure (image pull error, resource limits, probe failure, etc.).

### 4f. (Optional but recommended) Clear stale sessions before testing readiness

If nodes report `"ready": false"` due to all slots being occupied by old/orphaned sessions:
```
kubectl delete pod -n composer-user-workloads -l app=selenium-node-chrome
```
This is safe — the Deployment immediately recreates fresh pods with zero active sessions.

---

## Phase 5 — Verify the Deployed Grid

### 5a. Port-forward the hub service to your local machine

```
kubectl port-forward svc/selenium-hub 4445:4444 -n composer-user-workloads
```
Wait for confirmation output:
```
Forwarding from 127.0.0.1:4445 -> 4444
Forwarding from [::1]:4445 -> 4444
```
> Use a port other than 4444 locally (e.g. 4445) if you also have a local Docker Compose Selenium stack running — otherwise you'll get "address already in use."

### 5b. In a second terminal, check Grid status

```
curl http://localhost:4445/status
```
Expect `"ready": true`, both nodes `"availability": "UP"`, correct `"version"` string, and (on a clean/idle Grid) `"session": null` in all slots.

### 5c. Check the actual probe path used by Kubernetes' liveness/readiness probes

```
curl -i http://localhost:4445/wd/hub/status
```
Expect `HTTP/1.1 200 OK`. This confirms the Grid config surface (endpoint paths) still works correctly on the new version — this is NOT the same path as `/status`, and both should be checked.

### 5d. Run the functional client test against the k8s-deployed Grid

Update `test_grid.py`'s `command_executor` URL to match your port-forward's local port (e.g. `4445`):
```python
driver = webdriver.Remote(
    command_executor="http://localhost:4445/wd/hub",
    options=options,
)
```
Run:
```
python3 test_grid.py
```
Expect:
```
Page title: Example Domain
Current URL: https://example.com/
```

---

## Phase 6 — Validate the Real Workload (Not Just the Smoke Test)

The generic `example.com` test only proves basic Grid connectivity. It does **not** prove the actual crawler works — Chrome/driver behavior can change materially across large version jumps.

1. Trigger or observe a real DAG run (e.g. `avc_portal_import_daily__*`, `linked_login__*`) against the upgraded Grid via the Airflow/Composer UI.
2. Check the task logs for successful completion, especially around known fragile points (account-switcher, element lookups, login verification).
3. Compare against a pre-upgrade baseline run if one was captured.

To find Airflow task logs via `kubectl` (if not using the UI directly):
```
kubectl get pods -n composer-user-workloads
```
(Composer/Airflow worker pods may be in a different namespace or have different labels than expected — check with `kubectl get pods -A | grep -i worker` if unsure.)

---

## Phase 7 — Additional Issue Checklist Items (Not Infra-Specific)

These are separate from the deployment itself but were part of the original CVE-remediation issue:

- [ ] Re-run `snyk container test` against the new image tags and confirm 0 critical CVEs (the actual point of the version bump).
- [ ] Add a `docker` ecosystem block to `.github/dependabot.yml` so image tags are tracked going forward.
- [ ] Add `snyk container test` for both images into the `snyk_scan` nox session in `noxfile.py`.

---

## Phase 8 — Get the Code Changes Into Git / PR

**Important distinction:** applying changes via `kubectl apply` updates the LIVE cluster directly from your local files — it does NOT commit or push anything to git. These are two separate actions.

1. Confirm current git state:
   ```
   git status
   git log --oneline -5
   git branch --show-current
   ```

2. If not already on a feature branch, create one:
   ```
   git checkout -b fix/<issue-number>-selenium-grid-image-bump
   ```

3. Stage only the relevant files:
   ```
   git add docker-compose.yml
   git add kubernetes_deployments/selenium-hub-deployment.yaml
   git add kubernetes_deployments/selenium-node-chrome-deployment.yaml
   git diff --staged
   ```

4. Commit:
   ```
   git commit -m "Bump Selenium Grid images <OLD_VERSION> -> <NEW_VERSION> (#<issue-number>)"
   ```

5. Push and open a PR:
   ```
   git push origin <branch-name>
   ```

6. In the PR description, include:
   - The baseline comparison table (old vs. new, all checks)
   - What's verified vs. not yet verified (be honest — don't overstate)
   - A note on the readiness/liveness probe path if relevant
   - Rollback instructions (see below)

---

## Rollback Procedure

If something breaks after deployment:

**Quick rollback via kubectl:**
```
kubectl rollout undo deployment/selenium-hub -n composer-user-workloads
kubectl rollout undo deployment/selenium-node-chrome -n composer-user-workloads
```

**Or re-apply the old manifests directly:**
```
kubectl apply -f <old-version-of-the-yaml-file>.yaml -n composer-user-workloads
```

**Or delete and let it be recreated by whatever process manages it:**
```
kubectl delete -f kubernetes_deployments/selenium-hub-deployment.yaml -n composer-user-workloads
kubectl delete -f kubernetes_deployments/selenium-node-chrome-deployment.yaml -n composer-user-workloads
```

---

## Known Gotchas (Learned This Run)

1. **Port conflicts:** if you have a local Docker Compose Selenium stack running on `4444` AND try to `kubectl port-forward` the cluster's hub to the same local port, you'll get "address already in use." Use a different local port for the port-forward (e.g. `4445`), or stop the local Compose stack first.

2. **`/status` vs `/wd/hub/status`:** these are different paths. `/status` is a general Grid status check; `/wd/hub/status` is specifically what the Kubernetes liveness/readiness probes hit. Test both — passing one does not guarantee the other passes.

3. **`"ready": false"` doesn't always mean broken:** if all node slots are occupied by active sessions (e.g. from a running crawler DAG or leftover sessions from a previous test), the Grid reports `"ready": false"` even though every node is healthy (`"availability": "UP"`) — it just means no free capacity right now. Check the `slots[].session` field — `null` means free, a populated object means occupied.

4. **Kubernetes' `httpGet` probes only check the HTTP status code**, not the JSON response body. A `200 OK` with `"ready": false"` in the body still counts as a passing probe from Kubernetes' perspective. This is expected/correct behavior — you don't want a busy-but-healthy pod getting killed.

5. **After redeploying the hub, nodes need to re-register** — this happens automatically via the event bus, but expect a few seconds of `"ready": false", "nodes": []` (no error) immediately after a hub restart.

6. **A version bump alone doesn't guarantee stability** — a past incident on this same infrastructure (4.0 → 4.1.0) caused `tab crashed` errors that were only resolved by ALSO increasing container resource limits (CPU/memory), not just changing the image tag. If you see crash-type errors after a version bump, check resource limits before assuming the image itself is broken.

7. **`kubectl apply` reads from your local file, not from git.** The cluster can be running your updated code before it's ever committed or pushed — don't assume "it's deployed" means "it's in the repo," and vice versa.

---

## Quick Reference — Full Command Sequence (Copy/Paste)

```bash
# --- Local baseline (old version) ---
docker compose config --services
docker images | grep selenium
docker compose up -d selenium-hub chrome
docker compose ps
docker compose logs selenium-hub --tail=100
docker compose logs chrome --tail=100
curl http://localhost:4444/status
python3 test_grid.py

# --- Edit image tags in the 3 files, then ---

# --- Local re-validation (new version) ---
docker compose down selenium-hub chrome
docker compose up -d selenium-hub chrome
docker compose ps
curl http://localhost:4444/status
curl -i http://localhost:4444/wd/hub/status
python3 test_grid.py

# --- Cluster auth ---
gcloud container clusters get-credentials <cluster_name> --region <region> --project <project_id>

# --- Check current cluster state ---
kubectl get deployment selenium-hub -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get deployment selenium-node-chrome -n composer-user-workloads -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get pods -n composer-user-workloads
kubectl get svc -n composer-user-workloads

# --- Deploy ---
kubectl apply -f kubernetes_deployments/selenium-hub-deployment.yaml -n composer-user-workloads
kubectl apply -f kubernetes_deployments/selenium-node-chrome-deployment.yaml -n composer-user-workloads
kubectl rollout status deployment/selenium-hub -n composer-user-workloads
kubectl rollout status deployment/selenium-node-chrome -n composer-user-workloads

# --- Verify ---
kubectl get pods -n composer-user-workloads
kubectl port-forward svc/selenium-hub 4445:4444 -n composer-user-workloads
# (in a second terminal)
curl http://localhost:4445/status
curl -i http://localhost:4445/wd/hub/status
python3 test_grid.py   # command_executor updated to point at :4445

# --- Git ---
git status
git checkout -b fix/<issue-number>-selenium-grid-image-bump
git add docker-compose.yml kubernetes_deployments/selenium-hub-deployment.yaml kubernetes_deployments/selenium-node-chrome-deployment.yaml
git commit -m "Bump Selenium Grid images <OLD> -> <NEW> (#<issue-number>)"
git push origin <branch-name>
```