# Update `<project>-deploy-aks` and Sync Argo CD

This composite GitHub Action automates the deployment pipeline for Kubernetes-based applications managed via GitOps. It performs the following steps:

1. **Validates inputs** — checks formats and allowed values for all parameters.
2. **Checks out the deploy repository** — clones the `<project>-deploy-aks` repository at its `main` branch.
3. **Updates `values.yaml`** — patches the `repository` and `tag` fields under `helm/<env>/<level>/<app>/values.yaml` with the new image data.
4. **Commits and pushes** — commits the change and pushes it to the `main` branch of the deploy repository (skipped if no change is detected).
5. **Installs Argo CD CLI** — downloads (with SHA-256 verification) and caches `argocd` v3.4.3.
6. **Logs in to Argo CD** — authenticates against the Argo CD server via gRPC-Web.
7. **Triggers sync** — runs `argocd app sync` with `--prune` for `<domain>/<app>`.
8. **Waits for health** — runs `argocd app wait` until the application reaches `Synced` + `Healthy` status or the timeout expires.

---

## Permissions

### `gh_token`

The GitHub token provided via the `gh_token` input must have **write access to the `main` branch** of the `<project>-deploy-aks` repository. This is required to:

- check out the repository (read), and
- commit and push the updated `values.yaml` (write).

> ⚠️ If branch protection rules are enabled on `main`, the token owner must also be granted **bypass privileges** or the protection rules must allow the bot/app to push directly.

---

## Inputs

| Input              | Required | Default | Description                                                                                                                     |
|--------------------|:--------:|:-------:|---------------------------------------------------------------------------------------------------------------------------------|
| `gh_token`         | ✅       |         | GitHub token with write permissions to the main branch of the `<project>-deploy-aks` repository.                               |
| `deploy_aks_repo`  | ✅       |         | Repository hosting the `<project>-deploy-aks`, in the format `<owner>/<repo>`.                                                  |
| `env`              | ✅       |         | Target environment. Allowed values: `dev`, `uat`, `prod`.                                                                       |
| `level`            | ✅       |         | Level folder under `helm/<env>/`. Allowed values: `ext`, `mid`, `top`.                                                          |
| `app`              | ✅       |         | Application folder under `helm/<env>/<level>/`.                                                                                  |
| `image`            | ✅       |         | Full image reference including version and digest. Format: `<registry>/<image>:<version>@<digest>`.                             |
| `author`           | ✅       |         | Git author name used for the commit that updates the deploy repository.                                                          |
| `domain`           | ✅       |         | Argo CD domain (project) of the application, used to trigger the sync (`<domain>/<app>`).                                       |
| `argo_cd_server`   | ✅       |         | Argo CD server URL without protocol (e.g. `argocd.example.com`).                                                                |
| `argo_cd_username` | ✅       |         | Argo CD username for authentication.                                                                                             |
| `argo_cd_password` | ✅       |         | Argo CD password for authentication.                                                                                             |
| `argo_cd_timeout`  | ❌       | `300`   | Timeout in seconds applied to both the sync and the health-wait operations.                                                      |

---

## Usage example

```yaml
jobs:
  deploy:
    runs-on: [self-hosted-job, uat]
    permissions:
      contents: read   # only the calling repo; deploy repo access is via gh_token
    steps:
      - name: Update deploy-aks and sync Argo CD
        uses: pagopa/mil-actions/update-deploy-aks@73bd024abeec34d2dc0ac43999301015e817a472 # 2.0.7
        with:
          gh_token: ${{ secrets.GIT_PAT }}
          author: idpay-gh-bot
          deploy_aks_repo: pagopa/idpay-deploy-aks
          env: uat
          level: top
          app: mcshared-datavault
          image: ghcr.io/pagopa/mcshared-datavault:3.2.5@sha256:130e722be2b73b38c06251490cbc555ae633da80c53b6eb192bc5326f2edea6d
          domain: idpay
          argo_cd_server: ${{ vars.ARGO_CD_SERVER }}
          argo_cd_username: ${{ secrets.ARGO_CD_USERNAME }}
          argo_cd_password: ${{ secrets.ARGO_CD_PASSWORD }}
          argo_cd_timeout: 600 # 10 minutes
```

---

## Image format

The `image` input must follow the format:

```
<registry>/<path>:<version>@<digest>
```

Example:

```
ghcr.io/pagopa/my-service:1.2.3@sha256:fb230f97ba87b393746accf167067e0b75d20d2de5ee7cf847d50d4361f92e5f
```

The action automatically extracts `repository`, `version`, `digest`, and `tag` (`<version>@<digest>`) from this string and uses them to update `values.yaml`.

---

## Notes

- The Argo CD CLI (`argocd` v3.4.3) is cached between runs using `actions/cache` to speed up subsequent executions.
- The CLI binary is verified via SHA-256 before installation.
- If `values.yaml` contains no actual changes (same image already deployed), the commit/push step is skipped gracefully.
- Both the sync and the wait steps use `--grpc-web` to support Argo CD servers behind HTTP/HTTPS proxies that do not support raw gRPC.
