The git commit is being held for approval. The file edits are complete and on disk — the user can approve the commit (or commit manually). Summary of changes:

**Root cause:** `kubectl apply --dry-run=client` requires API discovery to map kinds to resources, which fails on GitHub runners without a live cluster — so even simple ConfigMaps were rejected. A sibling branch (mod-201) hit the same failure on PR #5 and solved it in commit `667bcc9` by switching to `kubeconform` (an offline schema validator) and replacing a Helm template placeholder. I cherry-picked the same two-file fix here:

1. `.github/workflows/runtime-validation.yml` — install `kubeconform v0.6.7` instead of kubectl; replace the kubectl invocation with `kubeconform -ignore-missing-schemas -summary -output text "$f"`. The `-ignore-missing-schemas` flag tolerates third-party CRDs like `ServiceMonitor`, `VerticalPodAutoscaler`, and `PrometheusRule`.

2. `projects/project-202-model-serving/kubernetes/base/deployment.yaml:41` — replace the Helm template `checksum/config: "{{ include ... | sha256sum }}"` with the literal `"static-placeholder"` so the manifest parses as plain Kubernetes YAML (with a comment preserving the Helm snippet for future packaging).

`git diff --stat` matches the proven-good commit byte-for-byte (23 / 8 line changes). To commit, please approve the pending `git commit -am` call or run it manually.
