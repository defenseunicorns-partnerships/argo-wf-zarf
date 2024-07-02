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
