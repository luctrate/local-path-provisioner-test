# PR Preparation Plan: Storage Features for local-path-provisioner

## Code Review Feedback

### Issues to Fix Before PR

1. ~~**Duplicate quota-helper Dockerfiles**~~ **FIXED** — Scripts moved to `package/quota-helper/`, `deploy/quota-helper/` removed. `package/Dockerfile.quota-helper` is the single source of truth and COPYs scripts from `package/quota-helper/`.

2. **`deleteKustomizeDeployment` in `helpers.go` uses `--ignore-not-found` but the original in `pod_test.go` did not** — The version that was moved to `helpers.go` added `--ignore-not-found`, which is better behavior for cleanup. The original `pod_test.go` version didn't have it. Good change, but just noting the intentional behavior change.

### Minor / Nice-to-Have

3. **`kustomization.yaml` changes add both `labels` AND `commonLabels`** — Many existing testdata kustomizations already had `labels:` with `includeSelectors: true`. The diff adds `commonLabels: app: local-path-provisioner` on top. This is redundant — `commonLabels` and `labels` both inject labels. Since the other agent is restructuring e2e tests for parallelism, I won't recommend changing this now, but if both settings inject the same label it's just noise.

4. ~~**`IMPLEMENTATION-storage-features.md`**~~ **FIXED** — Deleted.

5. ~~**`examples/quota-dual/`**~~ **FIXED** — Deleted `examples/quota-dual/` and the obsolete `examples/quota/`. Replaced with three focused, documented examples:
   - `examples/capacity-budget/` — per-path `maxCapacity` budget
   - `examples/size-validation/` — per-SC `minSize`/`maxSize` bounds
   - `examples/xfs-quota/` — filesystem-level XFS project quota enforcement

### Things That Look Good

- **`provisioner.go`**: Clean feature additions. `CapacityTracker` with `TryAllocate`/`TryResize` avoids TOCTOU races. `initCapacityTracker()` rebuilds state on restart. Size validation is straightforward. PathEntry JSON backward-compat (plain strings vs objects) is well handled.
- **`resize_controller.go`**: Solid workqueue-based controller. Startup reconciliation handles crash recovery. Expand + shrink are distinct paths. `escapeJSONString` for annotation patches is safe. `isOurPV` correctly checks provisioner annotation.
- **RBAC changes**: Both `deploy/local-path-storage.yaml` and `templates/clusterrole.yaml` correctly add PVC `update/patch`, PVC status `update/patch`, events `create/patch`, and StorageClass `get/list/watch` — all needed by the resize controller.
- **ConfigMap**: `resize` script added consistently to all three locations (values.yaml, example-config, local-path-storage.yaml).
- **`allowVolumeExpansion: true`** on the StorageClass — needed for Kubernetes to allow PVC spec size increases.
- **Helm chart**: `configmap.yaml` correctly renders the new `resize` script. `values.yaml` documents `storageClassConfigs`.
- **CI workflow**: Adds quota-helper image build steps correctly.
- **README**: Documents all three capacity features clearly and concisely.
- **quota-helper scripts**: setup/teardown/resize handle XFS project quotas well. flock for concurrent access, proper cleanup with grep -v, mount option validation.
- **Test refactoring**: Moving shared helpers to `helpers.go` is clean. `quota_test.go` is comprehensive (enforce, backward-compat, teardown, multi-PVC, space-recovery, resize-expand, shrink-success, shrink-rejected).

---

## Recommended Commit Structure

### Commit 1: `refactor(test): extract shared test helpers and fix test infra`
**Files:**
- `test/helpers.go` (new — `testdataFile`, `createCmd`, `runCmd`, `deleteKustomizeDeployment`)
- `test/pod_test.go`:
  - Remove `testdataFile`, `deleteKustomizeDeployment`, `createCmd`, `runCmd` (moved to helpers.go)
  - Add `QUOTA_HELPER_IMAGE` kind-load block in `SetupSuite()`
  - Fix `time.Tick` → `time.NewTicker` in `TestPodWithSecurityContext` and `verifyProvisioningFailed`
- `test/config.go` (add `quotaConfig` struct, `QUOTA_HELPER_IMAGE` field on `testConfig`)
- Existing test `kustomization.yaml` changes (11 files):
  - Add `commonLabels: app: local-path-provisioner`
  - Add `name: local-path-provisioner` image entry (where applicable)
- `test/testdata/security-basic-traversal-path/`:
  - Rename `storageclass.yaml` → `storage-class.yaml` (delete old, add new)
  - Update kustomization.yaml to reference new filename

**Rationale:** Pure refactor + infra fixes, no feature behavior change. Moves shared functions so both `e2e` and `quota_e2e` build tags can use them.

### Commit 2: `feat: add per-PVC size validation (minSize/maxSize)`
**Files:**
- `provisioner.go`:
  - `StorageClassConfigData`: add `MinSize`/`MaxSize` string fields
  - `StorageClassConfig`: add `MinSize`/`MaxSize` `*resource.Quantity` fields
  - `canonicalizeStorageClassConfig()`: parse and validate minSize/maxSize
  - `provisionFor()`: add size validation block (early return if below min or above max)
  - Note: this commit moves `storage` variable declaration up in `provisionFor()` — commit 3 depends on this

**Test files:**
- `test/testdata/size-reject-below-min/` (new)
- `test/testdata/size-reject-above-max/` (new)
- `test/testdata/size-accept-within-bounds/` (new)
- `test/pod_test.go`: add `TestSizeValidation` test

### Commit 3: `feat: add per-path capacity budget tracking`
**Files:**
- `provisioner.go`:
  - `CapacityTracker` struct + methods (`Allocate`, `Release`, `GetAllocated`, `HasCapacity`, `TryAllocate`, `TryResize`)
  - `PathEntry` struct (JSON backward-compat: plain strings vs objects)
  - `NodePathMapData.ParsePaths()` method
  - `PathConfig` struct with `MaxCapacity`
  - `NodePathMap.Paths` type change: `map[string]struct{}` → `map[string]*PathConfig`
  - `getPathOnNode()` rewritten with capacity logic + `requestedBytes` parameter
  - `initCapacityTracker()` — rebuild state on restart
  - `provisionFor()`: deferred capacity release on failure
  - `deleteFor()`: capacity release on successful teardown and node-gone
  - `canonicalizeStorageClassConfig()`: use `ParsePaths()` and `PathConfig`
- `deploy/example-config.yaml`: add `storageClassConfigs` example (includes `minSize`/`maxSize` AND `maxCapacity` path objects — both features shown together in the example)

**Test files:**
- `test/testdata/capacity-accept-within/` (new)
- `test/testdata/capacity-exhaustion/` (new)
- `test/pod_test.go`: add `TestCapacityWithinBudget`, `TestCapacityExhaustion`, `verifyCapacityExhaustion`

**Ordering note:** Must come after commit 2 — uses `storage.Value()` for `requestedBytes`, which depends on the `storage` variable being moved up by commit 2.

### Commit 4: `feat: add resize controller for PVC expand and shrink`
**Files:**
- `resize_controller.go` (new — 615 lines, workqueue-based controller)
- `main.go`: start resize controller (`NewResizeController` + `go resizeCtrl.Run(ctx)`)
- `provisioner.go`:
  - Constants: `ActionTypeResize`, `envVolQuotaType`, `quotaTypeAnnotationKey`, `annRequestedSize`, `annResizeStatus`, `annResizeMessage`
  - `ResizeCommand` field on `ConfigData` and `Config`
  - `volumeOptions.QuotaType` field
  - `createHelperPod()`: add resize key to `keyToPathItems` (cap 2→3), add `envVolQuotaType` to env vars, add `PodFailed` branch with `extractPodFailureReason`
  - `createHelperPodWithOutput()` (new — for resize helper pods)
  - `getHelperPodOutput()`, `getHelperPodLogsString()`, `extractPodFailureReason()`, `truncateLogs()` (new helper functions)
  - `canonicalizeConfig()`: add `ResizeCommand`

**Deploy files:**
- `deploy/local-path-storage.yaml`: resize script in ConfigMap, `allowVolumeExpansion: true` on SC, RBAC changes (PVC update/patch, PVC/status update/patch)
- `deploy/example-config.yaml`: add resize script
- `deploy/chart/local-path-provisioner/values.yaml`: add resize script default
- `deploy/chart/local-path-provisioner/templates/configmap.yaml`: render resize script
- `deploy/chart/local-path-provisioner/templates/clusterrole.yaml`: RBAC for PVC patch, PVC/status update/patch

**Note:** This commit includes `envVolQuotaType`, `quotaTypeAnnotationKey`, and `volumeOptions.QuotaType` because `resize_controller.go` depends on them. They are quota-related constants but are structurally required by the resize controller.

### Commit 5: `feat: add XFS project quota enforcement via quota-helper`
**Files:**
- `package/quota-helper/` (new directory: helperPod.yaml, setup, teardown, resize scripts)
- `package/Dockerfile.quota-helper` (new — debian bookworm-slim + xfsprogs, COPYs scripts)
- `.github/workflows/build.yml` (quota-helper image build steps)
- `provisioner.go`:
  - `provisionFor()`: read `quotaEnforcement` from SC parameters, set `QuotaType` on volume options
  - `provisionFor()`: write `quotaTypeAnnotationKey` annotation on PV creation
  - `deleteFor()`: read `quotaType` from PV annotation for teardown
  - `createHelperPodWithOutput()`: add `envVolQuotaType` to env vars

**Test files:**
- `test/testdata/quota-unsupported-fs/` (new)
- `test/pod_test.go`: add `TestQuotaOnUnsupportedFilesystem` (e2e tag — tests rejection on non-XFS)

### Commit 6: `test: add XFS quota e2e tests`
**Files:**
- `test/quota_test.go` (new — build tag `quota_e2e`)
- `test/testdata/quota-base/` (new — base kustomization + config for quota tests)
- `test/testdata/quota-registry/` (new — in-cluster registry for image pushing)
- `test/testdata/quota-xfs-enforce/` (new)
- `test/testdata/quota-backward-compat/` (new)
- `test/testdata/quota-teardown-cleanup/` (new)
- `test/testdata/quota-multiple-pvcs/` (new)
- `test/testdata/quota-space-recovery/` (new)
- `test/testdata/quota-resize-expand/` (new)
- `test/testdata/quota-shrink-success/` (new)
- `test/testdata/quota-shrink/` (new)

### Commit 7: `docs: add README section for volume capacity features`
**Files:**
- `README.md`

### Commit 8: `chore: replace examples with focused capacity feature examples`
**Files:**
- `examples/quota/` (deleted — obsolete upstream example)
- `examples/capacity-budget/` (new — kustomization, pvc, README)
- `examples/size-validation/` (new — kustomization, storage-class, pvc, README)
- `examples/xfs-quota/` (new — kustomization, storage-class, pvc, README)

---

## Separability Notes

### provisioner.go split across commits 2–5
The `provisioner.go` changes must be committed in order (2→3→4→5) because:
1. **Commit 2→3**: Commit 3 uses `storage.Value()` which depends on the `storage` variable being declared earlier by commit 2
2. **Commit 3→4**: Resize controller references `CapacityTracker.TryResize`, `PathConfig`, `NodePathMap.Paths` — all introduced in commit 3
3. **Commit 4→5**: Quota constants (`envVolQuotaType`, `quotaTypeAnnotationKey`, `volumeOptions.QuotaType`) go in commit 4 because the resize controller won't compile without them. Commit 5 adds the WRITING side (setting quota type based on SC parameters)

### deploy/example-config.yaml split across commits 3–4
The `storageClassConfigs` block (with `minSize`/`maxSize` + `maxCapacity` paths) goes in commit 3 since it references capacity budget concepts. The `resize` script addition goes in commit 4.

### Each commit should compile
- After commit 1: tests refactored, no feature changes, compiles
- After commit 2: size validation works standalone, compiles
- After commit 3: capacity budget works standalone, `TryResize` exists but is unused, compiles
- After commit 4: resize controller works, quota constants exist but quota type is never set (resize passes empty string), compiles
- After commit 5: quota enforcement fully wired up, compiles

---

## Files to Exclude from PR

- `PLAN.md` — this file

---

## ~~Fix Required Before Committing~~ DONE

**FIXED**: Scripts moved to `package/quota-helper/` and `package/Dockerfile.quota-helper` now COPYs them into the image.

---

## Progress Notes

### Completed

- [x] **Dockerfile fix** — `package/Dockerfile.quota-helper` now COPYs scripts from `package/quota-helper/` and is the single source of truth for building the quota-helper image. `deploy/quota-helper/` (Dockerfile + scripts + helperPod.yaml) deleted entirely. CI workflow (`build.yml`) uses `context: ./` so the COPY paths resolve correctly with no workflow changes needed.

- [x] **Dev notes cleanup** — `IMPLEMENTATION-storage-features.md` deleted.

- [x] **Examples overhaul** — Deleted `examples/quota/` (obsolete upstream CentOS 7 example, no flock, no resize, no auto-detect) and `examples/quota-dual/` (dev scratchpad that duplicated scripts). Replaced with three clean, kustomize-based examples each with a README:
  - `examples/capacity-budget/` — demonstrates `maxCapacity` on a node path (200Gi budget)
  - `examples/size-validation/` — demonstrates `minSize`/`maxSize` on a StorageClass (1Gi–50Gi)
  - `examples/xfs-quota/` — demonstrates `quotaEnforcement: xfs` with full helperPod, privileged mounts, and delegating setup/teardown/resize to the quota-helper image

### Remaining

- [ ] **Commit structuring** — split working tree into the 8 commits outlined above

### Deferred (no action needed)

- **Item 2** — `--ignore-not-found` in `helpers.go` — intentional improvement
- **Item 3** — redundant `commonLabels` + `labels` in kustomization.yaml — deferred (parallel e2e restructure in progress)
