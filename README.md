# gitops

Repo GitOps pour bootstraper un cluster Kubernetes (Talos) avec Argo CD comme moteur de synchronisation et GCP Secret Manager comme source de vérité pour les secrets.

## Vue d'ensemble

```
GCP Secret Manager
      │
      │  (lu par)
      ▼
External Secrets Operator (ESO)  ──► crée des Secret K8s
      │
      │  (lus par)
      ▼
Argo CD Vault Plugin (AVP, type=kubernetessecret)
      │
      │  (rend les <path:...> dans les manifests)
      ▼
Argo CD ──► applique les Applications du repo
```

Les composants installés **manuellement** (hors GitOps) :

- **Cilium** comme CNI — [manuals/cillium/values.yaml](manuals/cillium/values.yaml) (config typée Talos : `kubeProxyReplacement: strict`, KubePrism via `localhost:7445`).
- **Argo CD** avec sidecars AVP — [manuals/argocd/avp-values.yaml](manuals/argocd/avp-values.yaml) (3 plugins : `avp`, `avp-all`, `avp-helm`).

Tout le reste est déployé par Argo CD via une **app-of-apps** pointée sur [infra/](infra/).

## Pré-requis

- Un cluster Talos prêt (control-plane + workers).
- `kubectl` configuré sur le cluster.
- `helm` installé localement.
- Un projet GCP avec **Secret Manager** activé et un compte de service ayant le rôle `roles/secretmanager.secretAccessor`. Récupérer la clé JSON du SA.
- Les secrets attendus présents dans GCP Secret Manager (cf. [§ Secrets attendus](#secrets-attendus)).

## Installation pas à pas

### 1. Installer Cilium (CNI)

```bash
helm repo add cilium https://helm.cilium.io
helm repo update

helm install cilium cilium/cilium \
  --namespace kube-system \
  --values manuals/cillium/values.yaml
```

Vérifier :

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
cilium status   # si la CLI cilium est installée
```

### 2. Installer Argo CD avec AVP

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm install argocd argo/argo-cd \
  --namespace argocd \
  --values manuals/argocd/avp-values.yaml
```

Le chart déploie en plus :

- un `ServiceAccount` `argocd-repo-server` + RBAC pour lire les Secrets dans le namespace `argocd` (utilisé par AVP en mode `kubernetessecret`),
- un `ConfigMap` `cmp-plugin` avec les 3 plugins (`avp`, `avp-all`, `avp-helm`).

Récupérer le mot de passe initial admin :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

### 3. Créer le secret GCP pour ESO

ESO sera installé par Argo CD à l'étape suivante, mais le `ClusterSecretStore` ([post-install-cr/eso.yaml](post-install-cr/eso.yaml)) référence un Secret `gcp-sa-secret` dans le namespace `external-secrets`. Il faut le créer avant que les `ExternalSecret` se réconcilient.

```bash
kubectl create namespace external-secrets

kubectl -n external-secrets create secret generic gcp-sa-secret \
  --from-file=secret-access-credentials=/chemin/vers/sa-key.json
```

> Le `ClusterSecretStore` cible le projet GCP `amonnier` — adapter dans [post-install-cr/eso.yaml](post-install-cr/eso.yaml) si besoin.

### 4. Créer le projet Argo `infra`

Toutes les Applications référencent `project: infra`. À créer une fois :

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra
  namespace: argocd
spec:
  description: Infra & apps bootstrap
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: '*'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
EOF
```

### 5. Lancer le bootstrap (app-of-apps)

Appliquer une `Application` racine qui pointe sur le dossier [infra/](infra/) :

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
spec:
  project: infra
  source:
    repoURL: https://github.com/monnierant/gitops
    path: infra
    targetRevision: HEAD
    plugin:
      name: avpall
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

À partir de là, Argo CD synchronise en cascade :

| Étape                                                                        | Application       | Rôle                                                                                                       |
| ---------------------------------------------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------------------------- |
| `infra/eso.yaml`                                                             | `eso`             | Installe External Secrets Operator                                                                         |
| `infra/post-install-cr.yaml` → [post-install-cr/](post-install-cr/)          | `post-install-cr` | Crée le `ClusterSecretStore` GCPSM + middleware Traefik                                                    |
| `infra/argo-secret.yaml` → [argo-secrets/](argo-secrets/)                    | `argo-secrets`    | `ExternalSecret` qui peuplent `infra-secrets` et la clé SSH du repo privé                                  |
| `infra/external-repos.yaml` → [external-repos/](external-repos/)             | `external-repos`  | Branche le repo `private-gitops` (clé SSH)                                                                 |
| `infra/longhorn.yaml`                                                        | `longhorn`        | Storage                                                                                                    |
| `infra/smbc-driver.yaml`                                                     | `csi-smb`         | CSI SMB                                                                                                    |
| `infra/traefik.yaml` + `infra/traefic-rbac.yaml` + `infra/argo-ingress.yaml` | `traefik`         | Ingress + dashboard Argo CD sur `argocd.amonnier.fr`                                                       |
| `infra/security.yaml` → [security/](security/)                               | `security`        | FastAPI security helper                                                                                    |
| `infra/apps-inventory.yaml` → [apps-inventory/](apps-inventory/)             | `apps-inventory`  | Catalogue d'Applications utilisateur (Authentik, n8n, MCP, …) qui pointent à leur tour vers [apps/](apps/) |

## Secrets attendus

Les secrets suivants doivent exister dans GCP Secret Manager (projet `amonnier`) :

| Clé GCPSM                           | Utilisation                                                                                                                                                                   |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `k8s1-infra`                        | Bag de secrets pour l'infra : `email`, `entrypoint-ip`, `traefik-node-name`, … référencés via `<path:argocd:infra-secrets#...>` dans [infra/traefik.yaml](infra/traefik.yaml) |
| `k8s1-github-private-gitops-deploy` | Deploy key SSH (clé privée) du repo `monnierant/private-gitops`                                                                                                               |

Côté Kubernetes (créés par ESO) :

- `argocd/infra-secrets` — clé/valeur réutilisée par AVP via `<path:argocd:infra-secrets#email>` etc.
- `argocd/private-gitops-deploy-secrets` — `kubernetes.io/ssh-auth`.
- `argocd/argocd-private-gitops-ssh` — `Secret` typé `argocd.argoproj.io/secret-type: repository` consommé directement par Argo CD.

## Comment AVP rend les secrets

AVP tourne en sidecar du `repo-server` avec `AVP_TYPE=kubernetessecret`. Les manifests utilisent la syntaxe :

```yaml
stringData:
  password: "<path:argocd:infra-secrets#email>"
```

Au moment du `manifest generation`, AVP lit le Secret `infra-secrets` du namespace `argocd` (peuplé par ESO depuis GCP Secret Manager) et substitue la valeur. Les trois plugins disponibles :

- **`argocd-vault-plugin`** — applique AVP à tout `*.yaml` (hors `*values.yaml` et `*/statics/*`) du dossier ciblé.
- **`avpall`** — variante : applique AVP à tous les `*.yaml` du dossier (hors `*values.yaml` et `*/statics/*`).
- **`argocd-vault-plugin-helm`** — pour les charts Helm locaux : `helm template | argocd-vault-plugin generate -`.

### Où placer le plugin

Le plugin s'attache **à l'Application qui pointe le dossier**, pas au fichier qui contient les `<path:...>`. Conséquence : pour qu'un `<path:...>` soit rendu dans `infra/traefik.yaml`, c'est l'Application racine `bootstrap` (qui lit `infra/`) qui doit déclarer le plugin — l'Application `traefik` elle-même n'a rien à faire.

| Cas                                                                          | Où déclarer `source.plugin.name`                                                             |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Raw YAML avec `<path:...>` dans un dossier                                   | sur l'Application qui pointe ce dossier                                                      |
| Application Helm (chart distant) avec `<path:...>` inline dans `helm.values` | sur l'Application **parente** (celle qui lit le `*.yaml` contenant cette Application enfant) |
| Chart Helm local + `<path:...>` dans `values.yaml`                           | `argocd-vault-plugin-helm` sur l'Application                                                 |

Exemple — Application avec plugin explicite :

```yaml
spec:
  source:
    repoURL: https://github.com/monnierant/gitops
    path: infra
    targetRevision: HEAD
    plugin:
      name: avpall # ou argocd-vault-plugin-helm
```

## Accès au dashboard

Une fois Traefik et le certificat Let's Encrypt provisionnés : <https://argocd.amonnier.fr> (cf. [infra/argo-ingress.yaml](infra/argo-ingress.yaml)).

## Mise à jour

Renovate est branché sur le repo ([renovate.json](renovate.json)) — il ouvre des PR pour les `targetRevision` Helm et les images Docker référencées.
