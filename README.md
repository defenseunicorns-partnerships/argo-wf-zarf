# argo-wf-zarf
Zarf package for Argo Workflows

## Prerequisites
* UDS CLI: > v0.10.3
* Docker
* K3d
* [Argo CLI](https://argo-workflows.readthedocs.io/en/latest/walk-through/argo-cli/)
* Standard Linux tools like `rm`, `cp`, etc

If trying to work with the `ironbank` flavor, you will need the following environment variables set:
* `REGISTRY1_USERNAME`: username for registry1.dso.mil
* `REGISTRY1_PASSWORD`: token provided by registry1.dso.mil

## Installation
Most of the installation can be done via the tasks.yaml and `uds run` commands.  Here are some good targets to start with:
* `ci:up`: A clean installation including a k3d cluster and uds-slim-dev installation
* `tests:template-up`: Deploys a hello world WorkflowTemplate
* `tests:submit-workflow`: Submits a hello world workflow using kubectl
* `tests:server-token` : Gets a JWT for the server-api client
* `tests:submit-api-workflow --set JWT={JWT}` Submits a workflow using the argo server (replace `{JWT}` with the actual token)
* `ci:package-recycle`: Removes the package, rebuilds it, and redeploys it.
* `ci:down`: removes everything and deletes the k3d cluster
* `clean`: Cleans the directory of all build/test artifacts
* `test`: Runs the full end-to-end test suite

Run `uds run --list-all` to see all available tasks.  Of note, if building your own zarf package, you need to include the `--components=dev-setup` flag in the `zarf package create` command in order to use minio vice AWS S3 storage.

## Argo Server
This deployment sets up a read-only installation of the Argo Server and exposes the host at `https://workflows.uds.dev` by default (actually `https://workflows.{DOMAIN}`).  AuthService integration is a little spotty so you may need to navigate to user info and click login to trigger the redirects.

## Workflow specifics
The `/test/workflows/` directory has a hello-world template and workflow to test the deployment.  If a workflow is failing you can elect to keep the pod alive to look at logs by adding the field `spec.podGC.strategy` and setting it to `OnWorkflowSuccess` (the deployment defaults it to `OnPodCompletion`).

This deployment sets the `controller.workflowRestrictions.templateReferencing` to `Secure`.  This means any `Workflow` or `CronWorkflow` resource MUST reference a `WorkflowTemplate` deployed to the namespace in order for the controller to run the workflow.

## S3 Key gotcahs
If using AWS S3 as an artifact repository, do not use a leading `/` to define S3 keys.

If using Minio S3 as an artifact repository, use a leading `/` only for `input` artifacts.  Do not use a leading `/` for `output` artifacts.

## Using Argo Server with keycloak
Anything trying to connect to the Argo Server, must have a keycloak-issued JWT with the `argo-server` audience.  Additionally, the `client_id` must be `server-api` to use `POST`, `PUT`, or `DELETE` methods.

Programmatically, a token can be retrieved by the following:
```bash
# Get the client secret
CLIENT_SECRET="$(uds zarf tools kubectl get secret -n argo argo-workflows-client -o=jsonpath='{.data.secret}' | base64 --decode)"
JWT = "$(
  curl -X 'POST' \
    'https://sso.uds.dev/realms/uds/protocol/openid-connect/token' \
    -d grant_type=client_credentials \
    -H 'accept: application/json' \
    -H "Authorization: Basic $(echo -n server-api:${CLIENT_SECRET} | base64)" | jq -r '.access_token'
)
```
An example command to submit workflows:
```bash
curl -i -X 'POST' \
  'https://workflows.uds.dev/api/v1/workflows/argo/submit' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${JWT}" \
  -d '{
  "resourceName": "hello-world-template",
  "resourceKind": "WorkflowTemplate",
  "submitOptions": {
      "generateName": "hello-world-",
      "parameters": []
    }
  }'
```

## Zarf variables

| variable name                | required | default                   | description                                                                                                                                       |
| -----------------------------| -------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| ARCHIVE_TTL                  | no       | "10d"                     | Time to keep archived workflows in postgres before garbage collection.                                                                            |
| ARGO_REGISTRY                | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| ARGO REPO                    | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| ARGO_SERVER_REPO             | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| CLIENT_SECRET                | no       | argo-workflows-client     | The secret where the the clientSecret and other info about the server-api keycloak client reside                                                  |
| CONTR_CPU_LIM                | yes      | 1200m                     | CPU Limit for the Workflow Controller                                                                                                             |
| CONTR_MEM_LIM                | yes      | 2Gi                       | Memory Limit for the Workflow Controller                                                                                                          |
| CONTR_CPU_REQ                | yes      | 500m                      | CPU Request for the Workflow Controller                                                                                                           |
| CONTR_MEM_REQ                | yes      | 1Gi                       | Memory Request for the Workflow Controller                                                                                                        |
| DEFAULT_ARTIFACT_REPO        | yes      | minio-artifact-repository | choice between minio-artifact-repository for a dev setup with minio, or aws-artifact-repository for an AWS S3 setup with IRSA                     |
| DEPLOY_POSTGRESQL            | no       | None                      | If not set to False, deploys the postgresql chart to the argo namespace                                                                           |
| DEV_DEPLOYMENT               | yes      | None                      | Boolean value (set in zarf-config.yaml) on whether to use static credentials for artifact repositories                                            |
| DOMAIN                       | no       | uds.dev                   | The Domain for the deployment (for virtualService)                                                                                                |
| EXEC_CPU_LIM                 | yes      | 500m                      | CPU Limit for the Workflow Executor                                                                                                               |
| EXEC_MEM_LIM                 | yes      | 1Gi                       | Memory Limit for the Workflow Executor                                                                                                            |
| EXEC_CPU_REQ                 | yes      | 100m                      | CPU Request for the Workflow Executor                                                                                                             |
| EXEC_MEM_REQ                 | yes      | 512Mi                     | Memory Request for the Workflow Executor                                                                                                          |
| IRSA_ROLE_ARN                | no       | None                      | ARN of the role to assume if using IRSA                                                                                                           |
| PARALLELISM                  | yes      | 5                         | Number of workflows that can run in parallel                                                                                                      |
| PG_DB                        | no       | None                      | Name of the postgres database to archive workflows to                                                                                             |
| PG_HOST                      | no       | None                      | URL to create connection string to postgres database for workflow archiving                                                                       |
| PG_PASSWORD                  | no       | None                      | Admin password for postgres user                                                                                                                  |
| PG_PORT                      | no       | None                      | Port to connect to postgres on (typically 5432)                                                                                                   |
| PG_USER                      | no       | None                      | User to associate the archive database with                                                                                                       |
| PG_USER_PASSWORD             | no       | None                      | Password for the user associated with the archive database                                                                                        |
| PG_STORAGE_CLASS             | no       | null                      | Storage class for the PVC (used in AWS to ensure gp3 PV)                                                                                          |
| PSQL_REGISTRY                | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| PSQL_REPO                    | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| PSQL_TAG                     | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| RATE_BURST                   | yes      | 30                        | Burst QPS before the RATE_LIMIT is enforced                                                                                                       |
| RATE_LIMIT                   | yes      | 20                        | Limit Queries per Seconds (QPS) enforced at the controller                                                                                        |
| S3_ACCESS_KEY                | no       | None                      | Access key for access / secret pattern for auth to S3                                                                                             |
| S3_BUCKET_NAME               | yes      | None                      | Name of the S3 bucket to save artifacts to                                                                                                        |
| S3_ENDPOINT                  | yes      | None                      | Endpoint to connect to s3 "minio.test.dev" / "s3.amazonaws.com" etc.                                                                              |
| S3_PORT                      | yes      | None                      | Port to connect to S3 on (used to build network policies).  443 for AWS S3, usually 9000 for minio                                                |
| S3_REGION                    | no       | None                      | Used with AWS S3 to specify the region the S3 bucket is in                                                                                        |
| S3_SECRET_KEY                | no       | None                      | Secret key for access / secret pattern for auth to S3                                                                                             |
| SERVER_CPU_LIM               | yes      | 500m                      | CPU Limit for the Workflow Server                                                                                                                 |
| SERVER_MEM_LIM               | yes      | 128Mi                     | Memory Limit for the Workflow Server                                                                                                              |
| SERVER_CPU_REQ               | yes      | 100m                      | CPU Request for the Workflow Server                                                                                                               |
| SERVER_MEM_REQ               | yes      | 64Mi                      | Memory Request for the Workflow Server                                                                                                            |
| WF_TTL_SECONDS_AFTER_SUCCESS | yes      | 2                         | Seconds to keep a Workflow CR in the cluster before archiving after success                                                                       |
| WF_TTL_SECONDS_AFTER_FAILURE | yes      | 3600                      | Seconds to keep a Workflow CR in the cluster before archiving after failure                                                                       |

---
