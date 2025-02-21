# ⛵ Cluster Template - 5 Node Bare Metal Cluster

Welcome to my opinionated and extensible template for deploying a single Kubernetes cluster. The goal of this project is to make it easier for people interested in using Kubernetes to deploy a cluster at home on bare-metal or VMs. This template closely mirrors my personal [home-ops](https://github.com/onedr0p/home-ops) repository.

At a high level this project makes use of [makejinja](https://github.com/mirkolenz/makejinja) to read in a [configuration file](./config.sample.yaml) which renders out templates that will allow you to install and manage your Kubernetes cluster with.

## ✨ Features

The features included will depend on the type of configuration you want to use. There are currently **2 different types** of **configurations** available with this template.

1. **"Flux cluster"** - a Kubernetes cluster deployed on-top of [Talos Linux](https://github.com/siderolabs/talos) with an opinionated implementation of [Flux](https://github.com/fluxcd/flux2) using [GitHub](https://github.com/) as the Git provider and [sops](https://github.com/getsops/sops) to manage secrets.

    - **Required:** Some knowledge of [Containers](https://opencontainers.org/), [YAML](https://yaml.org/), and [Git](https://git-scm.com/).
    - **Components:** [flux](https://github.com/fluxcd/flux2), [cilium](https://github.com/cilium/cilium), [cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), and [reloader](https://github.com/stakater/Reloader).

2. **"Flux cluster with Cloudflare"** - An addition to "**Flux cluster**" that provides DNS and SSL with [Cloudflare](https://www.cloudflare.com/). [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) is also included to provide external access to certain applications deployed in your cluster.

    - **Required:** A Cloudflare account with a domain managed in your Cloudflare account.
    - **Components:** [ingress-nginx](https://github.com/kubernetes/ingress-nginx/), [external-dns](https://github.com/kubernetes-sigs/external-dns) and [cloudflared](https://github.com/cloudflare/cloudflared).

**Other features include:**

- Dev env managed w/ [mise](https://mise.jdx.dev/)
- Workflow automation w/ [GitHub Actions](https://github.com/features/actions)
- Dependency automation w/ [Renovate](https://www.mend.io/renovate)
- Flux HelmRelease and Kustomization diffs w/ [flux-local](https://github.com/allenporter/flux-local)


## 📣 Flux w/ Cloudflare post installation

### 🌐 Public DNS

> [!TIP]
> Use the `external` ingress class to make applications public to the internet.

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `flux-webhook` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

### 🏠 Home DNS

> [!TIP]
> Use the `internal` ingress class to make applications private to your network. If you're having trouble with internal DNS resolution check out [this](https://github.com/onedr0p/cluster-template/discussions/719) GitHub discussion.

`k8s_gateway` will provide DNS resolution to external Kubernetes resources (i.e. points of entry to the cluster) from any device that uses your home DNS server. For this to work, your home DNS server must be configured to forward DNS queries for `${cloudflare.domain}` to `${cloudflare.gateway_vip}` instead of the upstream DNS server(s) it normally uses. This is a form of **split DNS** (aka split-horizon DNS / conditional forwarding).

... Nothing working? That is expected, this is DNS after all!


## 🛠️ Talos and Kubernetes Maintenance

### ⚙️ Updating Talos node configuration

> [!IMPORTANT]
> Ensure you have updated `talconfig.yaml` and any patches with your updated configuration. In some cases you **not only need to apply the configuration but also upgrade talos** to apply new configuration.

```sh
# (Re)generate the Talos config
task talos:generate-config
# Apply the config to the node
task talos:apply-node IP=? MODE=?
# e.g. task talos:apply-node IP=10.10.10.10 MODE=auto
```

### ⬆️ Updating Talos and Kubernetes versions

> [!IMPORTANT]
> Ensure the `talosVersion` and `kubernetesVersion` in `talconfig.yaml` are up-to-date with the version you wish to upgrade to.

```sh
# Upgrade node to a newer Talos version
task talos:upgrade-node IP=?
# e.g. task talos:upgrade-node IP=10.10.10.10
```

```sh
# Upgrade cluster to a newer Kubernetes version
task talos:upgrade-k8s
# e.g. task talos:upgrade-k8s
```

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Flux itself! Merging a PR will cause Flux to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](./.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state. Most of these steps do not include a way to fix the problem as the problem could be one of many different things.

1. Verify the Git Repository is up-to-date and in a ready state.

    ```sh
    flux get sources git -A
    ```

    Force Flux to sync your repository to your cluster:

    ```sh
    flux -n flux-system reconcile ks flux-system --with-source
    ```

2. Verify all the Flux kustomizations are up-to-date and in a ready state.

    ```sh
    flux get ks -A
    ```

3. Verify all the Flux helm releases are up-to-date and in a ready state.

    ```sh
    flux get hr -A
    ```

4. Do you see the pod of the workload you are debugging?

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

5. Check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    ```

6. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

7. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on a NFS server. If you are unable to figure out your problem see the support sections below.

## 🧹 Tidy up

Once your cluster is fully configured and you no longer need to run `task configure`, it's a good idea to clean up the repository by removing the [templates](./templates) directory and any files related to the templating process. This will help eliminate unnecessary clutter from the upstream template repository and resolve any "duplicate registry" warnings from Renovate.

1. Tidy up your repository:

    ```sh
    task template:tidy
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: tidy up :broom:"
    git push
    ```

## 👉 Community Support

- Make a post in this repository's Github [Discussions](https://github.com/onedr0p/cluster-template/discussions).
- Start a thread in the `#support` or `#cluster-template` channels in the [Home Operations](https://discord.gg/home-operations) Discord server.

### DNS

Instead of using [k8s_gateway](https://github.com/ori-edge/k8s_gateway) to provide DNS for your applications you might want to check out [external-dns](https://github.com/kubernetes-sigs/external-dns), it has wide support for many different providers such as [Pi-hole](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/pihole.md), [UniFi](https://github.com/kashalls/external-dns-unifi-webhook), [Adguard Home](https://github.com/muhlba91/external-dns-provider-adguard), [Bind](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/rfc2136.md) and more.

### Storage

You might find you need persistent storage for your workloads with features like replicated storage or to connect to a NFS/SMB/iSCSI server. If you need any of those features be sure to check out the projects like [rook-ceph](https://github.com/rook/rook), [longhorn](https://github.com/longhorn/longhorn), [openebs](https://github.com/openebs/openebs), [democratic-csi](https://github.com/democratic-csi/democratic-csi), [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs), [csi-driver-smb](https://github.com/kubernetes-csi/csi-driver-smb)
or [synology-csi](https://github.com/SynologyOpenSource/synology-csi).

### Community Repositories

Community member [@whazor](https://github.com/whazor) created [Kubesearch](https://kubesearch.dev) to allow searching Flux HelmReleases across Github and Gitlab repositories with the `kubesearch` topic.
