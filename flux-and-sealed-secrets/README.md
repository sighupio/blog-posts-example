# Secrets management with FluxV2 using Bitnami Sealed Secrets

This repository contains a quick demo on secrets management with Bitnami Sealed Secrets in a Flux workflow.

## Prerequisites

To follow this tutorial, you need:

- FluxCLI `>= 0.14` ([installation guide][flux-installation])
- Minikube `>= v1.20` ([installation guide][minikube-installation])
- Kubeseal `=v0.16.0` ([installation guide][kubeseal-installation])
- GitHub account

## 0. Create GitHub Token and setup environment variables

Create a GitHub personal access token following [this guide][github-token-guide]. Remember to grant all `repo` permissions.

Once created, export the following environment variables:

```bash
export GITHUB_TOKEN=<TOKEN_ID>
export GITHUB_USER=<MY_GITHUB_USER> (e.g. nikever)
```

## 1. Start Minikube cluster

At the root level of this repository, execute:

```bash
export REPO_DIR=$(PWD)
```

Start a minikube cluster:

```bash
cd $REPO_DIR/minikube
make setup
```

By default, the command spins up a one node Kubernetes cluster of version `v1.19.4` in a VirtualBox VM (CPUs=2, Memory=4096MB, Disk=20000MB).

You can pass additional parameters to change the default values. Please referer to this [Makefile](minikube/Makefile) for further details on the cluster creation.

## 1. Install FluxV2

1. Open a terminal and bootstrap Flux with `fluxcli`:

```bash
flux bootstrap github \
    --owner $GITHUB_USER \
    --repository demo-flux-secrets-fleet \
    --branch main \
    --path ./minikube-cluster \
    --personal
```

The `bootstrap` command:

- creates a `demo-flux-secrets-fleet` repository
- generates Flux components manifests
- commits Flux components manifests to the `main` branch
- installs the Flux components in the `flux-system` namespace
- configures the target cluster to synchronize with the `./minikube-cluster` path inside the repository

2. While you wait for the commands to complete, you can verify that the pods in the `flux-system` are becoming running:

```bash
kubectl get pods -n flux-system -w
```

3. Clone the `demo-flux-secrets-fleet` repository

```bash
git clone https://github.com/$GITHUB_USER/demo-flux-secrets-fleet
cd demo-flux-secrets-fleet/
```

## 2. Deploy Hello secret App with Flux

1. Create a `hello-secret` folder inside the `minikube-cluster` folder:

```bash
mkdir hello-secret
```

2. Create a Flux `GitRepository` manifest pointing to the [Hello Secret][hello-secret-repository] repository's master branch:

```bash
flux create source git hello-secret \
    --url https://github.com/nikever/kubernetes-hello-secret \
    --branch main \
    --interval 1m \
    --export \
    > ./hello-secret/hello-secret-source.yaml
```

3. Create a Flux `Kustomization` manifest to build and apply the kustomize directory located in the Hello Secret repository under the `manifests` folder.

```bash
flux create kustomization hello-secret \
  --source=hello-secret \
  --path="./manifests" \
  --prune=true \
  --interval=1m \
  --export \
  > ./hello-secret/hello-secret-kustomization.yaml
```

4. Deploy via GitOps

```bash
git add -A && git commit -m "Deploy Hello Secret App"
git push
```

5. Wait for Flux to reconcile everything:

```bash
watch flux get sources git
watch flux get kustomizations
```

> You can force the reconciliation with `flux reconcile source git flux-system`

6. Check the resources deployed:

```bash
kubectl get all -n hello
```

Output:

```bash
NAME                             READY   STATUS                       RESTARTS   AGE
pod/hello-app-6cb6ddd95f-2chmq   0/1     CreateContainerConfigError   0          54s

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/hello-svc   NodePort   10.103.216.81   <none>        8080:30964/TCP   54s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-app   0/1     1            0           54s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-app-6cb6ddd95f   1         1         0       54s
```

As you can see, the pod `pod/hello-app-6cb6ddd95f-2chmq` is in the `CreateContainerConfigError` status. To become running, the pod needs a `hello-secret` secret, which is currently not deployed:

```bash
...
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Warning  Failed     43s (x8 over 2m6s)  kubelet            Error: secret "hello-secret" not found
```

### 3. Deploy Bitnami sealed secrets

1. Create a `sealed-secrets` folder inside the `minikube-cluster` folder:

```bash
mkdir sealed-secrets
```

2. Create a `HelmRepository` manifest

```bash
flux create source helm sealed-secrets \
    --url https://bitnami-labs.github.io/sealed-secrets \
    --interval 1h \
    --export \
    > ./sealed-secrets/sealed-secrets-source.yaml
```

3. Create Helm release manifest:

```bash
flux create helmrelease sealed-secrets \
    --interval=1h \
    --release-name=sealed-secrets \
    --target-namespace=flux-system \
    --source=HelmRepository/sealed-secrets \
    --chart=sealed-secrets \
    --chart-version=">=1.15.0-0" \
    --crds=CreateReplace \
    --export \
    > ./sealed-secrets/sealed-secrets-release.yaml
```

4. Deploy via GitOps:

```bash
git add -A && git commit -m "Deploy Bitnami sealed secrets"
git push
```

5. Wait for Flux to reconcile everything:

```bash
watch flux get sources git
```

> You can force the reconciliation with `flux reconcile source git flux-system`

6. Check the resources deployed:

```
kubectl get pods -n flux-system | grep sealed-secrets
```

## 4. Create a sealed hello-secret

At startup, the sealed-secrets controller generates a 4096-bit RSA key pair and persists the private and public keys as Kubernetes secrets in the `flux-system` namespace.

1. Retrieve the public key with:

```bash
kubeseal --fetch-cert \
    --controller-name=sealed-secrets \
    --controller-namespace=flux-system \
    > pub-sealed-secrets.pem
```

The public key can be safely stored in Git and can be used to encrypt secrets without direct access to the Kubernetes cluster.

2. Create the secret manifest:

```bash
kubectl create secret generic -n hello hello-secret \
    --from-literal=secret=1234 \
    --dry-run=client \
    -o yaml > hello-secret.yaml
```

3. Seal the secret using `kubeseal`:

```bash
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
    < hello-secret.yaml \
    > hello-secret-sealed.yaml
```

4. Remove the plain secret and commit the sealed one to deploy it:

```bash
rm hello-secret.yaml
git add hello-secret-sealed.yaml && git commit -m "Add sealed hello-secret"
git push
```

5. Wait for the `SealedSecret` to be deployed:

```bash
kubectl get SealedSecret -n hello -w
```

> You can force the reconciliation with `flux reconcile source git flux-system`

## 5. Test the hello-secret app

The `hello-secret` app contains a containerized Go web server application that responds to all HTTP requests with the value of the `SECRET` environment variable.

1. Restart the `hello-app` pod:

```bash
kubectl delete pod -n hello $(kubectl get pods -n hello -o=jsonpath='{.items..metadata.name}')
```

2. Check that now is running:

```bash
kubectl get pods -n hello
```

Output:

```bash
NAME                         READY   STATUS    RESTARTS   AGE
hello-app-6cb6ddd95f-558ts   1/1     Running   0          36s
```

3. Check that the `hello-secret` app is finally working:

```bash
curl -H "Host: hello-secret.info" $(minikube ip)
```

Output:

```bash
Hello, secret!
Secret: 1234
```

## 6. Clean up

1. Delete the `minikube` cluster:

```bash
cd $REPO_DIR/minikube
make delete
```

2. Delete the `demo-flux-secrets-fleet` repository (optional).

## References

- [Get started with Flux][flux-get-started]
- [Flux installation][flux-installation]
- [Flux Sealed Secrets][flux-sealed-secrets]

[hello-secret-repository]: https://github.com/nikever/kubernetes-hello-secret

[flux-get-started]: https://fluxcd.io/docs/get-started/
[flux-installation]: https://fluxcd.io/docs/installation/
[flux-sealed-secrets]: https://fluxcd.io/docs/guides/sealed-secrets/

[minikube-installation]: https://minikube.sigs.k8s.io/docs/start/
[kubeseal-installation]: https://github.com/bitnami-labs/sealed-secrets/releases/tag/v0.16.0

[github-token-guide]: https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token
