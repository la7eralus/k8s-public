#### Talosctl Config
`export TALOSCONFIG=~/Downloads/github/k8s/talos/talosconfig`

#### Wipe/Reset Talos VM 
`talosctl reset --graceful=false --reboot`

## Initial Installation
#### Install VM
`talosctl apply-config --insecure --nodes 192.168.4.10 --file talos/controlplane.yml`

#### Bootstrap (after autoreboot) 
`talosctl bootstrap`

#### Bootstrap ArgoCD
`helm upgrade --install argocd argo/argo-cd --create-namespace --namespace argocd`

#### Import Sealed Secrets Key
`kubectl create -f talos/sealed-secrets.yml`

#### Bootstrap Sealed Secrets
`helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets --create-namespace --namespace sealed-secrets`

#### AppofApps 
`kubectl -n argocd create -f apps/appofapps/appofapps.yml -f apps/argocd/extra/argocd-k8s.yml`

#### Get ArgoCD admin user token
`kubectl -n argocd get secret argocd-initial-admin-secret -o yaml | grep pass | cut -d ":" -f 2 | sed -e "s/ //" | base64 -d`

#### Get Kubernetes Dashboard Token
`kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d`

## Manual Installation (base-cluster)
#### NFS Provisioner
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --create-namespace --namespace nfs-subdir-external-provisioner --values=apps/nfs-subdir-external-provisioner/values.yml
```

#### Sealed Secrets
```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets --create-namespace --namespace sealed-secrets --values=apps/sealed-secrets/values.yml
```

#### MetalLB
```
helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb --create-namespace --namespace metallb --values=apps/metallb/values.yml
kubectl -n metallb create -f apps/metallb/extra/metallb-k8s.yml
```

#### Traefik
```
helm repo add traefik https://traefik.github.io/charts
helm upgrade --install traefik traefik/traefik --create-namespace --namespace traefik --values=apps/traefik/values.yml
```

#### Kubernetes-dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard --values=apps/kubernetes-dashboard/values.yml
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```

#### Argo CD
```
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd --create-namespace --namespace argocd --values=apps/argocd/values.yml
kubectl -n argocd get secret argocd-initial-admin-secret -o yaml | grep pass | cut -d ":" -f 2 | sed -e "s/ //" | base64 -d
kubectl -n argocd create -f apps/appofapps/appofapps.yml
```

### Useful Commands 
#### Debug pod
`kubectl -n kube-system debug -it coredns-78f679c54d-k5zcw --image=busybox:1.28 --target=coredns`

#### Delete Failed and Succeeded pods
`kubectl get pods -A; kubectl delete pods --field-selector status.phase=Failed -A; kubectl delete pods --field-selector status.phase=Succeeded -A; kubectl get pods -A`

#### Create sealed secret (namespace is important, importing from different namespace won't work)
`kubectl create secret generic cf-creds -n traefik --from-literal=CF_API_EMAIL='user@example.com' -o yaml | kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets -o yaml > cf-creds.yml`

#### Encrypt with sops (public age Key)
`sops --encrypt --encrypted-regex '^(content|tls.crt|tls.key|crt|key|secret|secretboxEncryptionSecret|token|id|ca|data)$' --age age1x... talos/controlplane.yml > talos/controlplane.enc.yml`

#### Decrypt with sops (age key in ~/.config/sops/age/keys.txt)
`sops --decrypt talos/controlplane.enc.yml > talos/controlplane.yml`

#### Export sealed secrets key
`kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml >sealed-secrets.yml`

#### Upgrade talos
`talosctl upgrade --preserve --nodes 192.168.4.10 --image factory.talos.dev/installer/ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515:v1.6.0`

#### Upgrade k8s
`talosctl --nodes 192.168.4.10 upgrade-k8s --to 1.29.0`

## Todo: 
- argocd-notifications
- Maybe: https://xphyr.net/post/ocp_syno_csi/#defining-the-synology-storage-class
