# Introduction
## Mission objective
This repository contains everything you need to install a Siderolabs Talos cluster with various Application Workloads from scratch. The whole Bootstrapping is heavily tested, and is working pretty fine for me.  
I took care of properly encrypting all existing secrets, but I want to point out that this is only intended for use in private scenarios.
My main target is to have everything as code in this repo (yes, also all needed secrets). The only true private thing thats left is my sops [age](https://github.com/FiloSottile/age) key, which I use to encrypt the talos config files.

## Repo structure
`talos` (Folder): Contains the main talos [machine config](https://www.talos.dev/v1.7/reference/configuration/) for controlplane and worker nodes. Also contains the talosconfig file, which is needed to communicate with the talos instances.

`apps` (Folder): Contains Helm values and extras aswell as plain Kubernetes Ressources. It's mainly consumed by argocd. The `attic` Folder is an archive for old, unused stuff.

`renovate.js` (File): Main renovate repo config file.

## Overall Architecture
My Talos Kubernetes Cluster runs in a Proxmox Cluster. The following Specs are suitable for a single schedulable controlplane node. They can be reduced a lot, if you plan to add dedicated worker nodes.
- 12GB Ram
- 4 vCPUs
- 1 Disk with 20GB (Talos System - also stores needed Container Images)
- 1 Disk with 10GB (Longhorn Storage - depends heavily on planned Data usage of your application workload)
- UEFI Bios with Secure Boot and an additional TPM
  - Supported since [Sidero 1.7.0](https://github.com/siderolabs/talos/issues/8131#issuecomment-2081548700)

Talos shares a high available virtual IP Adress between all controlplane nodes that can be used to talk to the kubernetes cluster via kubectl. Please note that this virtual IP Adress should not be used for talosctl communication. This should be done via the assigned Virtual Machine IP (e.g. reserved IP via dhcp). For convenience, I set a DNS Entry (talos.s3rv.dev) that points to this IP.

HTTPS Ingress to the cluster is handled via MetalLB and Traefik, which utilises a service of type `Loadbalancer`. The IP Adress of this service can be set to static in the Traefik Service (`loadBalancerIP`). I also set a wildcard DNS Entry (*.s3rv.dev) that points to the MetalLB IP Adress. Cert Manager creates my Let's Encrypt certificate for this Wildcard Entry.

## System applications I use (in deployment order)
#### [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
ArgoCD is my GitOps Tool of choice, it takes care that all desired manifests are actually deployed to the kubernetes cluster. I decided to go the app of apps route, because it simplifies the deployment a lot, is prone to errors and self healing. 

It works as follows:  
I have an app of apps application (`apps/appofapps/appofapps.yml`), that spins up all my other applications (`apps/appofapps/templates/*.yml`). 
These applications make use of ArgoCDs multisource applications. This means that one application can consist of ressources that are located in different repos. This for example is the case when I want to use an official upstream helm chart with my own values file (`apps/appofapps/templates/traefik.yml`). 

All `*-k8s.yml` files contain plain Kubernetes Manifests. I use them when no suitable helm chart is available or when I want to apply extra Manifests in addition to the helm chart (unfortunately not all Helm charts support extra manifests in their values file). The `-k8s` suffix is also interpreted by renovate.  
ArgoCD sends me telegram notifications for successful and failed deployments, if the label `notification: "true"` is found on the ArgoCD application.

#### [Sealed Secrets](https://sealed-secrets.netlify.app/)
The Sealed Secrets Controller is the one key piece that enables me to store my secrets in a public repository. The sealed secret can be created by sending a kubernetes secret to the sealed secrets controller via kubeseal, where it is encrypted with a private keypair. This sealed secret can then be safely stored in a public repository.  
When it is applied to the cluster it is decrypted by the sealed secrets controller which then creates a corresponding kubernetes secret.  
Since the only thing that can decrypt the sealed secret is the private keypair in the sealed secrets namespace, its a good idea to keep a safe backup of it.  
Please note that sealed secrets are namespaced resources (at least in the default config), so they can only be decrypted in the namespace that they were created in.

#### [MetalLB](https://metallb.org/)
MetalLB is a Software Loadbalancer implementation for Kubernetes. It can be very handy, if you want to route various traffic to your Kubernetes Cluster.
Basically you don't need MetalLB, if you only have plain HTTP/S traffic with no special requirements. In this scenario, the service type `ClusterIP` with an `externalIP` entry should suffice.  
The reason I choose to use MetalLB is a somewhat special Usecase, because I also host my DNS Server ([AdGuard Home](https://adguard.com/en/adguard-home/overview.html)) in Kubernetes. Since I want to be able to view the real IP of the requesting clients (for statistics and host based rules), I need to set the `externalTrafficPolicy` Option in the service. However, this is only possible in a service of type `Loadbalancer`, which is only working if you have a Loadbalancer implementation (like MetalLB).  
My MetalLB configuration is very simple and it basically just consists of two Manifest: One IP Adress Pool with 2 IPs (one for Traefik, one for DNS) and a L2 advertisment, that announces the IPs in my network. 

#### [Cert Manager](https://cert-manager.io/)
The Cert Manager Operator is a fully automated certificate issuer. You just need to provide a CRD of type `Issuer` (which in my case is cloudflare), and a CRD of type `Certificate`. Cert Manager then sets the TXT record requested by the ACME server (Let's Encrypt) on the domain root (`_acme-challenge.s3rv.dev`) and informs the ACME server that the domain is ready for verification. The certificate is stored in a secret in the traefik namespace where it is used as `defaultCertificate`.

#### [Traefik](https://doc.traefik.io/traefik/)
Traefik is my Ingress Controller and is used to route HTTP/S traffic to the right pod in my cluster. I choose it because it has a well maintained and very flexible Helm chart. Traefik is constantly evolving and one of my favourite CNCF Projects.  
Besides disabling the metrics and traefik port and redirecting web to websecure there is not much special going on here. The `IngressRoute` CRDs are created in the corresponding namespaces and picked up by Traefik. The reverse proxied Hosts outside the cluster are configured via `File` Provider.

#### [Longhorn](https://longhorn.io/)
Storage in Kubernetes is always tricky. For me it was important that my storage solution is simple, supports data locality, is replicated/high available, encrypted and allows scheduled system backups. All this is possible with longhorn. I attached a second disk to my talos node, which is exclusively used by longorn. I also had to add two talos system extensions ([reference](https://longhorn.io/docs/1.6.1/advanced-resources/os-distro-specific/talos-linux-support/)).  
In order to [encrypt](https://longhorn.io/docs/1.6.1/advanced-resources/security/volume-encryption/) the volume I had to create a secret in the longhorn namespace, that contains important encryption metadata like `CRYPTO_KEY_CIPHER` and `CRYPTO_KEY_HASH` as well as the actual passphrase  `CRYPTO_KEY_VALUE`. The Storage class Manifest that is also created with the longhorn application references this secret. I also added a `CronJob` Ressource that creates nightly backups to my external NFS Server, and deletes backups that are older than 3 days. The PVs and PVCs in the various Namespaces can be created via the Longhorn UI which means that they are also included in the system backup. This comes very handy if you want to reinstall your cluster from scratch because you don't have to even think about these ressources. All you have to do is restoring a system backup and you are good to go. The downside of this is that the used Helm charts have to support something like the `existingClaim` or `persistentVolumeClaim` field.

#### [Renovate](https://docs.renovatebot.com/)
Renovate is used to keep all applications up to date. At 5pm it checks wether there are any new releases and creates a pull request acordingly. This Pull Request is then assigned to me so that I have the chance to review the proposed changes. All open Renovate Pull Requests are automatically merged at 4am, unless they are Major or Longhorn updates (these have to be explicitly approved). This settings are configured in the renovate.json that can be found in the repository root.
 
# Detailed Steps
## Prepare Reset (only if you have already a running instance)
#### Talosctl Config
`export TALOSCONFIG=~/Downloads/github/k8s/talos/talosconfig`

#### Create longhorn Systembackup
```
cat <<EOF | kubectl create -f -
apiVersion: longhorn.io/v1beta2
kind: SystemBackup
metadata:
  name: reinstall
  namespace: longhorn-system
spec:
  volumeBackupPolicy: always
EOF
```

#### Wipe/Reset Talos VM 
`talosctl reset --graceful=false --reboot`

## Installation
#### Install Controlplane
`talosctl apply-config --insecure --nodes 192.168.4.6 --file talos/controlplane.yml`

#### Bootstrap (after autoreboot)
`talosctl bootstrap --nodes 192.168.4.6`

#### Install Worker if wanted (Change persistent hostname in config per worker)
`talosctl apply-config --insecure --nodes 192.168.4.7 --file talos/worker.yml`

#### Bootstrap ArgoCD, Import Sealed Secrets Key, Bootstrap Sealed Secrets
```
helm repo add argo https://argoproj.github.io/argo-helm && \
helm upgrade --install argocd argo/argo-cd --create-namespace --namespace argocd -f apps/argocd/values.yml && \
kubectl create -f talos/sealed-secrets.yml && \
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets && \
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets --create-namespace --namespace sealed-secrets -f apps/sealed-secrets/values.yml
```

#### AppofApps 
`kubectl -n argocd create -f apps/appofapps/appofapps.yml -f apps/argocd/extra/argocd-k8s.yml`

#### Get ArgoCD admin user token
`kubectl -n argocd get secret argocd-initial-admin-secret -o yaml | grep pass | cut -d ":" -f 2 | sed -e "s/ //" | base64 -d`

#### Restore longhorn Systembackup
```
cat <<EOF | kubectl create -f -
apiVersion: longhorn.io/v1beta2
kind: SystemRestore
metadata:
  name: restore-after-reinstall
  namespace: longhorn-system
spec:
  systemBackup: reinstall
EOF
```

# Useful Commands 
#### Spawn Debug pod
`kubectl -n kube-system debug -it coredns-78f679c54d-k5zcw --image=busybox:1.28 --target=coredns`

#### Delete all failed and succeeded pods
`kubectl get pods -A; kubectl delete pods --field-selector status.phase=Failed -A; kubectl delete pods --field-selector status.phase=Succeeded -A; kubectl get pods -A`

#### Create sealed secret (namespace is important, importing from different namespace won't work)
`kubectl create secret generic cf-creds -n traefik --from-literal=CF_API_EMAIL='user@example.com' --from-literal=CF_PASSWORD='Pa55w0rd' -o yaml | kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets -o yaml > cf-creds.yml`

#### Encrypt with sops (public age Key)
`sops --encrypt --encrypted-regex '^(content|tls.crt|tls.key|crt|key|secret|secretboxEncryptionSecret|token|id|ca|data|environment)$' --age age1x... talos/controlplane.yml > talos/controlplane.enc.yml`

#### Decrypt with sops (age key in ~/.config/sops/age/keys.txt)
`sops --decrypt talos/controlplane.enc.yml > talos/controlplane.yml`

#### Export sealed secrets key
`kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets.yml`

#### Upgrade talos (controlplane)
`talosctl upgrade -n 192.168.4.6 --preserve -i factory.talos.dev/installer-secureboot/077514df2c1b6436460bc60faabc976687b16193b8a1290fda4366c69024fec2:v1.7.1`

#### Upgrade talos (worker)
`talosctl upgrade -n 192.168.4.7 -i factory.talos.dev/installer-secureboot/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.7.1`

#### Upgrade k8s
`talosctl --nodes 192.168.4.6 upgrade-k8s --to 1.29.0`

#### Force renew of LE cert
`cmctl renew s3rv-dev -n traefik`

#### Renew talosconfig cert (~/.talos/config)
`talosctl config new`

#### Renew kubeconfig (\~/.kube/config, NASA2:\~/kube/kubeconfig)
`talosctl -n 192.168.4.6 kubeconfig`
