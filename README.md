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
* `tests:package-up`: Deploys a hello world WorkflowTemplate
* `tests:submit-workflow`: Submits a hello world workflow
* `ci:package-recycle`: Removes the package, rebuilds it, and redeploys it.
* `ci:down`: removes everything and deletes the k3d cluster
* `clean`: Cleans the directory of all build/test artifacts
* `test`: Runs the full end-to-end test suite

Run `uds run --list-all` to see all available tasks.

## Workflow specifics
The `/test/workflows/` directory has a hello-world template and workflow to test the deployment.  If a workflow is failing you can elect to keep the pod alive to look at logs by adding the field `spec.podGC.strategy` and setting it to `OnWorkflowSuccess` (the deployment defaults it to `OnPodCompletion`).

The Zarf package that deploys the test WorkflowTemplate also deploys a pod called "wrapper" that has some CLI tools installed so that you can shell into it for testing purposes.

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
