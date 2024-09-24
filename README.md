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
* `tests:submit-workflow`: Submits a hello world workflow
* `ci:package-recycle`: Removes the package, rebuilds it, and redeploys it.
* `ci:down`: removes everything and deletes the k3d cluster
* `clean`: Cleans the directory of all build/test artifacts
* `test`: Runs the full end-to-end test suite

Run `uds run --list-all` to see all available tasks.  Of note, if building your own zarf package, you need to include the `--components=dev-setup` flag in the `zarf package create` command in order to use minio vice AWS S3 storage.

## Workflow specifics
The `/test/workflows/` directory has a hello-world template and workflow to test the deployment.  If a workflow is failing you can elect to keep the pod alive to look at logs by adding the field `spec.podGC.strategy` and setting it to `OnWorkflowSuccess` (the deployment defaults it to `OnPodCompletion`).

This deployment sets the `controller.workflowRestrictions.templateReferencing` to `Secure`.  This means any `Workflow` or `CronWorkflow` resource MUST reference a `WorkflowTemplate` deployed to the namespace in order for the controller to run the workflow.

## S3 Key gotcahs
If using AWS S3 as an artifact repository, do not use a leading `/` to define S3 keys.

If using Minio S3 as an artifact repository, use a leading `/` only for `input` artifacts.  Do not use a leading `/` for `output` artifacts.

## Using Argo Server with Client Auth
You will need a token to use the Argo Server with Client Auth.  A token can be created for an existing serviceaccount by creating a kubernetes secret with the following pattern:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wfapi.service-account-token
  namespace: argo
  annotations:
    kubernetes.io/service-account.name: wfapi
type: kubernetes.io/service-account-token
```
This assumes the service account named `wfapi` exists in the `argo` namespace and has appropriate permissions via a `rolebinding`.

Programmatically, the token can be retrieved by the following:
```bash
# Get the token
ARGO_TOKEN="$(uds zarf tools kubectl get secret -n argo wfapi.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"
```
An example command to list workflows:
```bash
## First, in a different terminal window, port-forward to the argo-server
kubectl port-forward svc/argo-workflows-server -n argo 2746:2746
## Then, in a separate terminal, run the following command
curl http://localhost:2746/api/v1/workflows/argo -H "Authorization: Bearer $ARGO_TOKEN"
```

## Zarf variables

| variable name                | required | default                   | description                                                                                                                                       |
| -----------------------------| -------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| ARCHIVE_TTL                  | no       | "10d"                     | Time to keep archived workflows in postgres before garbage collection.                                                                            |
| ARGO_REGISTRY                | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| ARGO REPO                    | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| ARGO_SERVER_REPO             | no       | None                      | Used with flavors to build the zarf package                                                                                                       |
| DEFAULT_ARTIFACT_REPO        | yes      | minio-artifact-repository | choice between minio-artifact-repository for a dev setup with minio, or aws-artifact-repository for an AWS S3 setup with IRSA                     |
| DEPLOY_POSTGRESQL            | no       | None                      | If not set to False, deploys the postgresql chart to the argo namespace                                                                           |
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
| WF_TTL_SECONDS_AFTER_SUCCESS | yes      | 2                         | Seconds to keep a Workflow CR in the cluster before archiving after success                                                                       |
| WF_TTL_SECONDS_AFTER_FAILURE | yes      | 3600                      | Seconds to keep a Workflow CR in the cluster before archiving after failure                                                                       |

---
