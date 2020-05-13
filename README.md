# Demo of the GitOps approach with Flux

Example configuration of GitOps. Tools used in this repo:
- [Flux](https://github.com/fluxcd/flux) - The GitOps Kubernetes operator
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) - A Kubernetes controller and tool for one-way encrypted Secrets
- [Traefik](https://github.com/containous/traefik) - The Cloud Native Edge Router
- [Kubeval](https://github.com/instrumenta/kubeval) - The validator for Kubernetes configuration files

## Prerequisites

You will need to have Kubernetes set up. For a quick local test, you can use [k3d](https://github.com/rancher/k3d), minikube, kubeadm or kind. Any other Kubernetes setup will work as well though.

## Step by step installation

1. Install `fluxctl`

```sh
# MacOS
brew install fluxctl

# Linux snap
sudo snap install fluxctl

# Linux pacman
pacman -S fluxctl

# Windows Chocolatey
choco install fluxctl
```

More info: https://docs.fluxcd.io/en/latest/references/fluxctl/

2. Modify arguments passed to Flux

In the `flux/deployment.yaml` file find corrensponding lines and modify it according to your desired configuration.

```
# URL to your repo
- --git-url=git@github.com:adam-golab/flux-demo
# Branch that you want to synchronize with the cluster
- --git-branch=master
# Tag name that will be used for labeling deployed changes
- --git-label=flux
# Your git username (used as an author of added tag)
- --git-user=adam-golab
# Your email (used as an author of added tag)
- --git-email=adam-golab@users.noreply.github.com
```

3. Deploy the Flux to your cluster

Point the file modified in the step above.

```sh
kubectl apply -f flux/deployment.yaml
```

And wait for Flux to start.

```sh
kubectl -n flux rollout status deployment/flux
```

4. Give write access to your repo

At startup Flux generates a SSH key and logs the public key. Find the SSH public key

```sh
fluxctl identity --k8s-fwd-ns flux
```

In order to sync your cluster state with git you need to copy the public key and create a deploy key with write access on your GitHub repository.

Open GitHub, navigate to your fork, go to **Setting > Deploy keys**, click on **Add deploy key**, give it a `Title`, check **Allow write access**, paste the Flux public key and click **Add key**. See the [GitHub docs](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys) for more info on how to manage deploy keys.

By default, Flux git pull frequency is set to 5 minutes. You can tell Flux to sync the changes immediately with:

```sh
fluxctl sync --k8s-fwd-ns flux
```

5. Prepare config for Traefik

I am keeping the config in the Kubernetes' Secret to not share it publicly. It is definitely a good practice when you want to limit accessing to secrets without loosing an ability to keep these in the git.

You can use the the template from the `traefik/config.template` directory. Just change the file name to `config.yaml` and provide your email for the Let's Encrypt notifications.

Remember to **not** commit this file into the repository, as we want to avoid sharing it publicly.

6. Create Sealed Secret with our config.

First create a Secret with the content of the `traefik/config.yaml` file.

```sh
kubectl create secret generic traefik-config --namespace=traefik --dry-run --from-file=config.yaml -o yaml >secret-config.yaml
```

Then we can encrypt the secret using Sealed Secret.

```sh
kubeseal -o yaml < secret-config.yaml > sealed-secret-config.yaml
```

Now you can commit **only** the `sealed-secret-config.yaml` file into the repository and verify if everything is deployed correctly.

7. Enjoy your deployed Hello World application via GitOps

### TODO

- add and describe tools for metrics (probably Prometheus and Grafana)
