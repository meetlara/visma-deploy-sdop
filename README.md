# visma-deploy-sdop

Registers each production deployment in Visma Hubble for the Engineering Performance (ENGP/SDOP) metrics.

It runs right after the deploy step of the deploy workflow:

- A successful deploy step is reported as Released, a failed one as Failed. If the deploy step never ran or was cancelled, nothing is reported.
- `deploymentDurationMinutes` runs from `deployment_start` (by default the start of the run attempt) to the end, plus 2 minutes of the ECS Blue/Green rollout.
- `totalLeadTimeMinutes` runs from the merge of the PR to the end, plus the same 2 minutes. A deploy that is re-run after a failure therefore carries the time the change waited.
- `numberOfChanges` is 1: one PR, one deployment. A failed deployment reports 0, because the run that releases it counts the change.
- `buildInfoUrl` links to the run attempt; the description starts with the PR link.
- Labels are `sub-repo-<repo>` and `sub-author-<author>`.

The step writes the Hubble link as a notice in the run summary, and exposes it as the `deployment_key` and `deployment_url` outputs.

## Usage

```yaml
- name: Log deployment start
  id: deployment_start
  run: echo "deployment_start=$(date +"%Y-%m-%dT%H:%M:%S%:z")" >> "$GITHUB_OUTPUT"

- name: Deploy to production
  id: deployment
  uses: meetlara/ecs-deploy@v3
  # ...

- name: Report the deployment to Visma SDOP
  uses: meetlara/visma-deploy-sdop@v3
  if: success() || failure()
  continue-on-error: true
  with:
    hubble_client_id: ${{ secrets.HUBBLE_CLIENT_ID }}
    hubble_client_secret: ${{ secrets.HUBBLE_CLIENT_SECRET }}
    deployment_status: ${{ steps.deployment.outcome }}
    deployment_start: ${{ steps.deployment_start.outputs.deployment_start }}
```

| Input | Default | |
|---|---|---|
| `hubble_client_id` / `hubble_client_secret` | | Visma Connect client with the `VismaHubbleTrackingApi:deploymentsWrite` scope |
| `application_external_id` | `1988` | Hubble product id of the application (it replaces the Jira component) |
| `deployment_status` | `success` | Outcome of the deploy step: `success` → Released, `failure` → Failed, anything else is skipped |
| `deployment_start` | start of the run attempt | When the deploy step started |
| `extra_labels` | | Extra labels (comma separated) |
| `test` | `false` | Excludes the deployment from the metrics |

## Hubble API

- Swagger: https://k8s.hubble.visma.com/tracking-system/api/swagger/index.html
- `POST /tracking-system/api/deployments` creates the deployment (status Started = 1).
- `PATCH /tracking-system/api/deployments/{externalId}/status` moves it to Released (2) or Failed (3).
- The token comes from `https://connect.visma.com/connect/token` with client credentials.
- Hubble's own action, [Hubble-ENGP-Reporter](https://github.com/Visma-Product-Security/Hubble-ENGP-Reporter), uses the same API. It counts every commit since the last successful deploy as a change; we keep one change per PR.
