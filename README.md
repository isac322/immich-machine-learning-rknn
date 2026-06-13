# immich-machine-learning-rknn

Automated rebuild of the [Immich](https://github.com/immich-app/immich) `prod-rknn` machine-learning image with [immich-app/immich#23958](https://github.com/immich-app/immich/pull/23958) ("perf: RKNN half RAM model usage + no weight duplication in multi-core") applied on top.

The PR keeps weights in shared memory across RKNN threads so `MACHINE_LEARNING_RKNN_THREADS >= 2` doesn't multiply NPU memory usage. On rk3588 boards this lets you raise threads without hitting the ~4 GiB IOMMU IOVA cap.

## Output

- Image: `ghcr.io/isac322/immich-machine-learning:<immich_tag>-rknn`
- Tag policy: identical to upstream `ghcr.io/immich-app/immich-machine-learning:<immich_tag>-rknn`
- Arch: `linux/arm64` (rk3588 / rk3576 / rk3568 / rk3566)
- Variant: `prod-rknn` (no `prod-cpu`, `prod-cuda`, `prod-openvino`, `prod-armnn`, `prod-rocm`)

## How it works

1. `.github/workflows/build.yaml` runs daily (cron) and on `workflow_dispatch`.
2. Resolves target Immich version (input or latest release).
3. Skips if `ghcr.io/isac322/immich-machine-learning:<tag>-rknn` already exists (unless `force_rebuild`).
4. Shallow-clones `immich-app/immich` at the target tag.
5. Applies every `patches/*.patch` file in sorted order (PR #23958 + local fixups on top).
6. If `git apply --check` fails → workflow fails loudly. No silent rollback. Refresh patch then retry.
7. Builds `machine-learning/Dockerfile` with `--target prod --build-arg DEVICE=rknn` on `ubuntu-24.04-arm` (GitHub-hosted ARM64 runner, free for public repos).
8. Pushes to `ghcr.io/isac322/immich-machine-learning:<tag>-rknn` with build cache stored at `:buildcache`.

## Refreshing the patch on upstream changes

### Patch list

| File | Purpose |
|---|---|
| `01-pr-23958-rknn-shared-weights.patch` | Upstream PR #23958 verbatim (rebased onto v2.7.5) |
| `02-fix-build-cross-use-venv-python.patch` | Local fixup: chain RUN commands with `&&` so build failures are not silently swallowed, and prepend `PATH=/opt/venv/bin` so `python3 -m pybind11 --includes` finds pybind11 inside the runtime venv |

When the workflow fails with "Patch does NOT apply" on a new Immich tag:

```bash
# Tag that broke us:
TAG=v2.8.0
git clone https://github.com/immich-app/immich.git immich
cd immich
git checkout "$TAG"
git fetch origin pull/23958/head:pr-23958
git apply --3way ../patches/01-pr-23958-rknn-shared-weights.patch  # try first
# If 3-way fails, manually merge PR #23958 into "$TAG":
git merge --no-ff --no-commit pr-23958      # resolve conflicts under machine-learning/
git diff --staged "$TAG" > ../patches/01-pr-23958-rknn-shared-weights.patch
cd ..
# Verify on a fresh clone:
rm -rf verify
git clone --depth 1 --branch "$TAG" https://github.com/immich-app/immich.git verify
git -C verify apply --check patches/01-pr-23958-rknn-shared-weights.patch
# Commit the refreshed patch, push, then re-trigger workflow.
```

## Why not fork Immich?

A full fork drifts: every upstream release bumps a thousand files we don't care about, and tracking with rebases costs time and merge skill we don't need to spend.

This repo carries **only** the patch file plus the rebuild pipeline. Upstream is read-only, our image always tracks an exact upstream tag, and our delta is one file you can read.

## Notes

- Initial package visibility is private. After the first successful build, set the package public via GitHub UI → Packages → `immich-machine-learning` → Package settings → Change visibility → Public. Subsequent rebuilds inherit that.
- Build cache lives at `ghcr.io/isac322/immich-machine-learning:buildcache`. Don't pull that as a runtime tag.
