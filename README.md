# argo-wf-zarf
Zarf package for Argo Workflows

## Prerequisites
* Zarf CLI: > v0.31.0
* UDS CLI: > v0.10.3

If trying to work with the `ironbank` flavor, you will need the following environment variables set:
* `REGISTRY1_USERNAME`: username for registry1.dso.mil
* `REGISTRY1_PASSWORD`: token provided by registry1.dso.mil

## Installation
Most of the installation can be done via the tasks.yaml and `uds run` commands.  Here are some good targets to start with:
* `workflows-up`: A clean install including a k3d cluster and uds-slim-dev installation
* `test-workflows`: Deploys a hello world template and submits a workflow for the template'
* `recycle-workflows`: Removes the package, rebuilds it, and redeploys it.  Of note, changes you make to `zarf-config.yaml` will be reflected but not saved to `test/zarf-config-dev.yaml`
* `cleanup`: removes everything and deletes the k3d cluster

## Workflow specifics
The `/test/workflows/` directory has a hello-world template and workflow to test the deployment.  If a workflow is failing you can elect to keep the pod alive to look at logs by adding the field `spec.podGC.strategy` and setting it to `OnWorkflowSuccess` (the deployment defaults it to `OnPodCompletion`).

If you want to include local images into the zarf package, it's recommended to setup a local registry.  There is a test container that includes the minio mc binary that can be built and pushed into the registry.  If you wish to deploy this, then change the `test-container` component in `zarf.yaml` to `required: true`.

You should also run the following targets with `uds run`:
* `registry-up`: Creates the local docker registry
* `build-image`: Runs the Dockerfile in `/test/workflows` and pushes it to the local registry

## Using Argo Server with Client Auth
The serviceaccount `wfapi` in the `argo` namespace has a token which can be found in the secret `wfapi.service-account-token`.  Programatically, it can be retrived by the following command:
```bash
ARGO_TOKEN="Bearer $(kubectl get secret jenkins.service-account-token -o=jsonpath='{.data.token}' | base64 --decode)"
```
An example command to list workflows:
```bash
curl http://argo-server.argo.svc.cluster.local:2746/api/v1/workspace/argo -H $ARGO_TOKEN
```
