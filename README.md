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

| Étape                                                                       | Application         | Rôle                                                                                                                                                             |
| --------------------------------------------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `infra/eso.yaml`                                                            | `eso`               | Installe External Secrets Operator                                                                                                                               |
| `infra/post-install-cr.yaml` → [post-install-cr/](post-install-cr/)         | `post-install-cr`   | Crée le `ClusterSecretStore` GCPSM + middleware Traefik                                                                                                          |
| `infra/argo-secret.yaml` → [argo-secrets/](argo-secrets/)                   | `argo-secrets`      | `ExternalSecret` qui peuplent `infra-secrets` et les clés SSH des repos privés                                                                                   |
| `infra/external-repos.yaml` → [external-repos/](external-repos/)            | `external-repos`    | Branche les repos privés `private-gitops` et `staging-gitops` (clés SSH)                                                                                         |
| `infra/longhorn.yaml`                                                       | `longhorn`          | Storage                                                                                                                                                          |
| `infra/metrics-server.yaml`                                                 | `metrics-server`    | Métriques instantanées (`kubectl top`, HPA). `--kubelet-insecure-tls` requis sur Talos                                                                           |
| `infra/monitoring.yaml` → [monitoring/](monitoring/)                        | `monitoring`        | Métriques historisées : kube-prometheus-stack bridé + Grafana. Cf. [ADR 0001](docs/adr/0001-metriques-historisees-sous-contrainte-disque.md)                     |
| `infra/smbc-driver.yaml`                                                    | `csi-smb`           | CSI SMB                                                                                                                                                          |
| `infra/traefic-rbac.yaml` + `infra/argo-ingress.yaml`                       | (RBAC + Ingress)    | RBAC Traefik + dashboard Argo CD sur `argocd.amonnier.fr` (Application `traefik` créée par `bootstrap-secrets`)                                                  |
| `infra/security.yaml` → [security/](security/)                              | `security`          | FastAPI security helper                                                                                                                                          |
| `infra/apps-inventory.yaml` → [apps-inventory/](apps-inventory/)            | `apps-inventory`    | Catalogue d'Applications utilisateur (Authentik, n8n, MCP, …) qui pointent à leur tour vers [apps/](apps/)                                                       |
| `infra/bootstrap-secrets.yaml` → [infra-with-secrets/](infra-with-secrets/) | `bootstrap-secrets` | App-of-apps **avec plugin `avpall`** pour tout ce qui contient des `<path:...>` (Traefik, avp-test). Retry jusqu'à ce que `infra-secrets` soit populated par ESO |

## Secrets attendus

Les secrets suivants doivent exister dans GCP Secret Manager (projet `amonnier`) :

| Clé GCPSM                           | Utilisation                                                                                                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `k8s1-infra`                        | Bag de secrets pour l'infra : `email`, `entrypoint-ip`, `traefik-node-name`, … référencés via `<path:argocd:infra-secrets#...>` dans [infra-with-secrets/traefik.yaml](infra-with-secrets/traefik.yaml) |
| `k8s1-github-private-gitops-deploy` | Deploy key SSH (clé privée) du repo `monnierant/private-gitops`                                                                                                                                         |
| `k8s1-github-staging-gitops-deploy` | Deploy key SSH (clé privée) du repo `MonniHermes/staging-gitops`, **en écriture** : l'Image Updater y pousse ses write-back Kustomize                                                                   |
| `k8s1-grafana`                      | JSON `{"admin-user": "...", "admin-password": "..."}` — consommé via `admin.existingSecret` par [monitoring/kube-prometheus-stack.yaml](monitoring/kube-prometheus-stack.yaml), sans passer par AVP     |

Côté Kubernetes (créés par ESO) :

- `argocd/infra-secrets` — clé/valeur réutilisée par AVP via `<path:argocd:infra-secrets#email>` etc.
- `argocd/private-gitops-deploy-secrets` — `kubernetes.io/ssh-auth`.
- `argocd/argocd-private-gitops-ssh` — `Secret` typé `argocd.argoproj.io/secret-type: repository` consommé directement par Argo CD.
- `argocd/argocd-staging-gitops-ssh` — idem pour `MonniHermes/staging-gitops`.

### Ajouter la deploy key d'un nouveau repo privé

Une deploy key GitHub est scopée à **un seul repo** : la clé d'un repo ne peut pas être réutilisée pour un autre, GitHub refuse l'enregistrement d'une clé publique déjà employée. Chaque repo privé branché sur Argo CD a donc sa propre paire et sa propre clé GCPSM.

Si le repo appartient à une **organisation**, vérifier d'abord que les deploy keys y sont autorisées — le réglage est org-wide et vaut `false` par défaut sur les orgs récentes, l'ajout de clé échoue alors en `HTTP 422: Deploy keys are disabled for this repository` :

```bash
gh api orgs/<ORG> --jq .deploy_keys_enabled_for_repositories
gh api -X PATCH orgs/<ORG> -f deploy_keys_enabled_for_repositories=true
```

```bash
ssh-keygen -t ed25519 -C "argocd@amonnier.fr" -f ./deploy-key -N ""

# Cle publique -> GitHub, en ecriture (--allow-write) si l'Image Updater
# doit pousser des write-back sur ce repo.
gh repo deploy-key add ./deploy-key.pub \
  --repo MonniHermes/staging-gitops --title "ArgoCD Write" --allow-write

# Cle privee -> GCP Secret Manager, sous le nom reference par l'ExternalSecret.
gcloud secrets create k8s1-github-staging-gitops-deploy \
  --project amonnier --data-file=./deploy-key

shred -u ./deploy-key ./deploy-key.pub
```

Puis ajouter l'`ExternalSecret` correspondant dans [argo-secrets/](argo-secrets/) et l'`Application` racine dans [external-repos/](external-repos/).

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

Le plugin s'attache **à l'Application qui pointe le dossier**, pas au fichier qui contient les `<path:...>`. Pour qu'un `<path:...>` soit substitué dans `infra-with-secrets/traefik.yaml`, c'est l'Application **parente** (`bootstrap-secrets`, qui lit `infra-with-secrets/`) qui déclare le plugin — l'Application `traefik` enfant elle-même n'a rien à faire.

| Cas                                                                          | Où déclarer `source.plugin.name`                                                             |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Raw YAML avec `<path:...>` dans un dossier                                   | sur l'Application qui pointe ce dossier                                                      |
| Application Helm (chart distant) avec `<path:...>` inline dans `helm.values` | sur l'Application **parente** (celle qui lit le `*.yaml` contenant cette Application enfant) |
| Chart Helm local + `<path:...>` dans `values.yaml`                           | `argocd-vault-plugin-helm` sur l'Application                                                 |

### Pourquoi bootstrap est splitté en deux

Chicken-and-egg : AVP a besoin du Secret `argocd/infra-secrets` pour résoudre les `<path:...>`, mais ce Secret est créé par ESO via `argo-secrets` — qui est lui-même déployé par `bootstrap`. Si `bootstrap` utilisait AVP, il échouerait à rendre ses manifestes au tout premier sync (le Secret n'existe pas encore) → aucune Application enfant ne se créerait → deadlock.

D'où le split :

- **`bootstrap`** — pas de plugin (mode `directory`, `recurse: false`). Lit `infra/*.yaml` (top-level uniquement), crée ESO + ExternalSecrets + Longhorn + … et l'Application `bootstrap-secrets`. Aucun de ces fichiers ne contient de `<path:...>`.
- **`bootstrap-secrets`** — plugin `avpall`. Lit `infra-with-secrets/*.yaml` (Traefik, avp-test, …). Tant que `infra-secrets` n'existe pas, son sync échoue et ArgoCD retry. Une fois ESO + ExternalSecret réconciliés, le sync passe. Pas de blocage des autres Applications pendant l'attente.

Tout fichier futur contenant des `<path:...>` doit aller dans [infra-with-secrets/](infra-with-secrets/), pas dans `infra/`.

## Accès au dashboard

Une fois Traefik et le certificat Let's Encrypt provisionnés : <https://argocd.amonnier.fr> (cf. [infra/argo-ingress.yaml](infra/argo-ingress.yaml)).

## Mise à jour

Renovate est branché sur le repo ([renovate.json](renovate.json)) — il ouvre des PR pour les `targetRevision` Helm et les images Docker référencées.
