# OpenGrok on Kubernetes

## Intro

OpenGrok is a fast source-code search and cross-reference tool that helps engineers navigate large codebases directly from a web interface. It indexes repositories and provides full-text search, symbol lookup, history, and code navigation, making it easier to investigate changes, understand dependencies, and find relevant implementation details across projects.

In this setup, OpenGrok runs on Kubernetes as a StatefulSet with persistent storage for source code, index data, configuration, and logs. A sidecar container periodically syncs Git repositories into the shared source directory, and the OpenGrok container indexes those repositories and serves the web UI on port 8080.

## Repository contents

This deployment includes:

* 1 StatefulSet: opengrok
* 2 containers in the same Pod:
    * opengrok: runs the OpenGrok web/indexing service
    * ubuntu: helper sidecar container that periodically clones or updates repositories under /opengrok/src
* 4 ConfigMaps:
    * `env-variables`: OpenGrok runtime and indexing environment variables
    * `init-script`: sidecar startup script that installs and starts cron
    * `repo-to-sync`: repository sync script
    * `configmap-logs`: OpenGrok Java logging configuration
* 2 Secrets:
    * `netrcsecret`: optional Git credentials mounted as /root/.netrc
    * `regcred`: optional image pull secret
* 2 Services:
    * `opengrok`: headless service required by the StatefulSet
    * `opengrok-clusterip`: external access service, currently configured as LoadBalancer
* 4 persistent volume claims created from volumeClaimTemplates:
    * `opengrok-src`: cloned source repositories
    * `opengrok-data`: OpenGrok index data
    * `opengrok-conf`: OpenGrok configuration data
    * `opengrok-logs`: OpenGrok log files

## How it works

The Pod contains the OpenGrok container and an Ubuntu sidecar container that share the same source-code volume.
```
StatefulSet: opengrok
        |
        |-- container: opengrok
        |     |
        |     |-- reads environment variables from the env-variables ConfigMap
        |     |-- reads logging configuration from configmap-logs
        |     |-- reads source code from /opengrok/src
        |     |-- writes index data to /opengrok/data
        |     |-- writes logs to /opengrok-logs
        |     |-- serves the OpenGrok UI on port 8080
        |
        |-- container: ubuntu
              |
              |-- starts /initscript/init-script.sh from the init-script ConfigMap
              |-- installs and starts cron
              |-- runs /opengrok_scripts/repoToSync.sh every 3 minutes
              |-- clones or updates repositories under /opengrok/src
              |-- can use /root/.netrc from netrcsecret for private Git authentication
```
The current repository sync script clones these public sample repositories:

* https://github.com/dockersamples/example-voting-app.git
* https://github.com/dockersamples/compose-dev-env.git

To index different repositories, update `configmap-sync-repo.yaml` and replace the repository URLs and target directories in `repoToSync.sh`.

## Important configuration notes

### Repository sync interval

The sidecar cron job currently runs every 3 minutes:
```
*/3 * * * * /opengrok_scripts/repoToSync.sh
```
This is configured inside configmap-init-script.yaml.

### OpenGrok reindex interval

OpenGrok reindexing is controlled separately through the REINDEX environment variable in configmap-env-variables.yaml:
```
REINDEX: "3"
```
This means the sidecar sync schedule and the OpenGrok reindex schedule are separate settings. In this repo they are both set to 3 minutes for quick testing.

### StorageClass

The PVC templates currently use the GKE StorageClass:
```
storageClassName: standard-rwo
```
This works for GKE. For another Kubernetes environment, update the storageClassName values in `statefulset.yaml` before applying the manifests.

### Secrets

The included secrets contain dummy values so the manifests can be applied as-is for demo/testing with public repositories.

For private Git repositories, replace the `.netrc` content in `secret-netrc.example.yaml` with real Git credentials.

For private container images, replace the Docker registry credentials in `secret-image-repo-credentials.example.yaml`. If all images are public and your cluster does not require an image pull secret, the imagePullSecrets reference can also be removed from the StatefulSet.

### Service exposure

The opengrok-clusterip service file is currently configured as a LoadBalancer service:
```
type: LoadBalancer
```
On GKE, this creates an external load balancer with an external IP. If you only need internal cluster access, change it back to ClusterIP.

## Deploy

Apply all manifests from the repository directory:
```
kubectl apply -f .
```
Check the created resources:
```
kubectl get all
kubectl get pvc
kubectl get svc
```
Wait for the StatefulSet Pod to be ready

## Access OpenGrok

Get the external IP if using the current LoadBalancer service:
```
kubectl get svc opengrok-clusterip
```
Open the UI in a browser: 
```
http://<EXTERNAL-IP>:8080
```
If the service is changed to ClusterIP, use port-forwarding instead:
```
kubectl port-forward svc/opengrok-clusterip 8080:8080
```
Then open: http://localhost:8080

## Useful commands

```
kubectl exec -it opengrok-0 -c opengrok -- /bin/sh

kubectl exec -it opengrok-0 -c ubuntu -- /bin/bash

kubectl logs opengrok-0 -c opengrok

kubectl logs opengrok-0 -c ubuntu

kubectl exec -it opengrok-0 -c ubuntu -- ls -la /opengrok/src

kubectl exec -it opengrok-0 -c ubuntu -- /opengrok_scripts/repoToSync.sh
```

## Cleanup

Delete all Kubernetes resources from this repo:
```
kubectl delete -f .
```
The StatefulSet creates PVCs from volumeClaimTemplates. Depending on the cluster and storage settings, PVCs may remain after deleting the StatefulSet. To remove the stored source, index, config, and log data, delete the PVCs manually:
```
kubectl get pvc
kubectl delete pvc <pvc-name>
```
Only delete PVCs if the stored OpenGrok data is no longer needed.
