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

## Initial Installation
#### Install Controlplane
`talosctl apply-config --insecure --nodes 192.168.4.6 --file talos/controlplane.yml`

#### Bootstrap (after autoreboot)
`talosctl bootstrap --nodes 192.168.4.6`

#### Install Worker (Change persistent hostname in config)
`talosctl apply-config --insecure --nodes 192.168.4.7 --file talos/worker.yml`

#### Bootstrap ArgoCD
```
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd --create-namespace --namespace argocd -f apps/argocd/values.yml
```

#### Import Sealed Secrets Key
`kubectl create -f talos/sealed-secrets.yml`

#### Bootstrap Sealed Secrets
```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets --create-namespace --namespace sealed-secrets -f apps/sealed-secrets/values.yml
```

#### AppofApps 
`kubectl -n argocd create -f apps/appofapps/appofapps.yml -f apps/argocd/extra/argocd-k8s.yml`

#### Get ArgoCD admin user token
`kubectl -n argocd get secret argocd-initial-admin-secret -o yaml | grep pass | cut -d ":" -f 2 | sed -e "s/ //" | base64 -d`

#### Get Kubernetes Dashboard Token
`kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d`

#### Restore longhorn System
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

## Useful Commands 
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

#### Upgrade talos (controlplane)
`talosctl upgrade -n 192.168.4.10 --preserve -i factory.talos.dev/installer/7d0c9bb43d29c3a66af593485d6d95ae14f241c90678c9cc05a4cf99d928ec1b:v1.6.3`

#### Upgrade talos (worker)
`talosctl upgrade -n 192.168.4.77 -i factory.talos.dev/installer/a7bcadbc1b6d03c0e687be3a5d9789ef7113362a6a1a038653dfd16283a92b6b:v1.6.3`

#### Upgrade k8s
`talosctl --nodes 192.168.4.10 upgrade-k8s --to 1.29.0`

#### Renew talosconfig cert (~/.talos/config)
`talosctl config new`

#### Renew kubeconfig (~/.kube/config, NASA2:~/kube/kubeconfig)
`talosctl -n 192.168.4.6 kubeconfig`