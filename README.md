# visma-deploy-sdop

Registers each production deployment in Visma Hubble for the Engineering Performance (ENGP/SDOP) metrics.

It runs as the last step of the deploy workflow, so the running workflow is the deployment:

- `deploymentDurationMinutes` and `totalLeadTimeMinutes` are the run duration plus 2 minutes of the ECS Blue/Green rollout.
- `numberOfChanges` is always 1: one PR, one deployment.
- `buildInfoUrl` links to the run; the description starts with the PR link.
- Labels are `sub-repo-<repo>` and `sub-author-<author>`.

Hubble creates every deployment as Started, so the action moves it to Released right after creating it.
Failed deployments are not reported.

## Usage

```yaml
- name: Report the deployment to Visma SDOP
  uses: meetlara/visma-deploy-sdop@v3
  continue-on-error: true
  with:
    hubble_client_id: ${{ secrets.HUBBLE_CLIENT_ID }}
    hubble_client_secret: ${{ secrets.HUBBLE_CLIENT_SECRET }}
```

| Input | Default | |
|---|---|---|
| `hubble_client_id` / `hubble_client_secret` | | Visma Connect client with the `VismaHubbleTrackingApi:deploymentsWrite` scope |
| `components` | `1988-Lara AI` | ENGP components (comma separated) that resolve to the Hubble application |
| `extra_labels` | | Extra labels (comma separated) |
| `test` | `false` | Adds `SDOPexclude` and excludes the deployment from the metrics |

## Hubble API

- Swagger: https://k8s.hubble.visma.com/tracking-system/api/swagger/index.html
- `POST /tracking-system/api/deployments` creates the deployment (status Started = 1).
- `PATCH /tracking-system/api/deployments/{externalId}/status` with `{"status": 2}` releases it (Failed = 3).
- The token comes from `https://connect.visma.com/connect/token` with client credentials.
