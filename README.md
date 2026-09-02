# opsctrl-actions

Shared GitHub Actions composite actions for OpsCtrl CI/CD. Consumed by
`opsctrl_platform_backend`, `opsctrl_portal` and `opsctrl_landing`.

Reference them pinned to a major tag:

```yaml
uses: Hillyon-Labs/opsctrl-actions/docker-build-push@v1
```

## Conventions

- **Image tags**: UAT builds are tagged `uat-<short-sha>` plus a moving `uat-latest`.
- **Charts** live inside each app repo under `charts/`; deploys layer
  `values.yaml` → `values-<env>.yaml` → secrets values (from a GitHub secret).
- **Cluster auth**: a base64 kubeconfig for a `github-deployer` ServiceAccount,
  stored as the `KUBE_CONFIG_UAT` environment secret.

## Actions

### `docker-build-push`

Builds with buildx and pushes to Docker Hub.

| input | required | notes |
|---|---|---|
| `image` | yes | e.g. `opsctrl/opsctrl_portal` |
| `tags` | yes | newline-separated tags |
| `build_args` | no | newline-separated `KEY=VALUE` (used for `NEXT_PUBLIC_*`) |
| `dockerhub_username` / `dockerhub_token` | yes | token needs read+write |
| `context` | no | default `.` |
| `platforms` | no | default `linux/amd64` |

### `setup-kube`

Installs helm + kubectl, writes the kubeconfig, exports `KUBECONFIG`, and
sanity-checks connectivity with `kubectl get nodes`.

| input | required | notes |
|---|---|---|
| `kubeconfig_b64` | yes | base64 kubeconfig |
| `helm_version` | no | default `v3.16.4` |
| `kubectl_version` | no | default `latest` |

### `helm-deploy`

`helm dependency build` + `helm upgrade --install --wait` with layered values.

| input | required | notes |
|---|---|---|
| `release` / `chart_path` / `namespace` | yes | |
| `values_files` | no | newline-separated, applied in order |
| `secrets_values_b64` | no | decoded to a temp file, applied last |
| `set` | no | newline-separated `key=value` (e.g. `image.tag=uat-abc1234`) |
| `timeout` | no | default `10m` |

## Example deploy job

```yaml
deploy-uat:
  runs-on: ubuntu-latest
  environment: uat
  concurrency: uat-deploy
  steps:
    - uses: actions/checkout@v4

    - id: meta
      run: echo "tag=uat-$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

    - uses: Hillyon-Labs/opsctrl-actions/docker-build-push@v1
      with:
        image: opsctrl/opsctrl_portal
        tags: |
          ${{ steps.meta.outputs.tag }}
          uat-latest
        dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
        dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}

    - uses: Hillyon-Labs/opsctrl-actions/setup-kube@v1
      with:
        kubeconfig_b64: ${{ secrets.KUBE_CONFIG_UAT }}

    - uses: Hillyon-Labs/opsctrl-actions/helm-deploy@v1
      with:
        release: opsctrl-portal
        chart_path: ./charts/opsctrl-portal
        namespace: opsctrl-system
        values_files: |
          ./charts/opsctrl-portal/values.yaml
          ./charts/opsctrl-portal/values-uat.yaml
        set: |
          image.tag=${{ steps.meta.outputs.tag }}
```

## Releasing changes

Commit to `main`, then move the `v1` tag:

```bash
git tag -f v1 && git push origin v1 --force
```
