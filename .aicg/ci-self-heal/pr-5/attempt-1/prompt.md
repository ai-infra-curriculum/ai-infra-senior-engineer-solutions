# Address CI failures on PR #5

## Goal

The PR you just opened failed CI. Fix the failures listed
below by editing files on the current branch. Do NOT regenerate
the content from scratch — make the minimal edit needed to
satisfy each failing check.

## Failed checks

### 1. `kubectl apply --dry-run=client` (failure)

- Details: <https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-solutions/actions/runs/27619491020/job/81664108321>
- Annotations:
  - `.github:2` (warning): Node.js 20 actions are deprecated. The following actions are running on Node.js 20 and may not work as expected: actions/checkout@v4, azure/setup-kubectl@v4. Actions will be forced to run with Node.js 24 by default starting June 16th, 2026. Node.js 20 will be removed from the runner on September 16t
  - `projects/project-202-model-serving/kubernetes/overlays/dev/dev-ingress.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/servicemonitor.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/serviceaccount.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/service.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/pvc.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/networkpolicy.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/hpa.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/deployment.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-202-model-serving/kubernetes/base/configmap.yaml:?` (failure): kubectl dry-run rejected manifest
  - `projects/project-201-distributed-training/kubernetes/ray-cluster.yaml:?` (failure): kubectl dry-run rejected manifest

## Output contract

- Edit ONLY files inside this repo on the current branch.
- Preserve the existing structure; do not delete sections.
- Do NOT touch CURRICULUM.md, README.md, or VERSIONS.md.
- One atomic commit covering all fixes is fine.
