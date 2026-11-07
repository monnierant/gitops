# Argo CD : montée de version (installation manuelle Helm)

Argo CD est installé **hors GitOps**, par `helm` et [avp-values.yaml](avp-values.yaml) (cf. [README racine](../../README.md#2-installer-argo-cd-avec-avp)). Le dossier `manuals/` n'est lu par aucune Application (l'app-of-apps `bootstrap` ne lit que `infra/*.yaml`, sans récursion) : merger ici ne déploie rien, toute montée passe par la procédure ci-dessous.

## État constaté (2026-11-07)

- Version servie : `v3.5.5` (`curl -s https://argocd.amonnier.fr/api/version`, sans authentification).
- Chart probable : **10.10.3**, préparé par le commit `30de18c` (« préparer la montée chart 10.10.3 (Argo CD 3.5.5) »). Les charts 10.10.5 et 10.11.0 embarquent aussi la v3.5.5 : seul `helm list -n argocd` (colonne `CHART`) fait foi.
- Les sidecars `avp-all` et `avp-helm` suivent la version du chart via `{{ .Chart.AppVersion }}`. Le sidecar `avp` et l'initContainer `download-tools` tournent sur `registry.access.redhat.com/ubi8`, sans tag fixe.

## Cible : dernier patch de la mineure 3.5

| | Avant | Cible |
|---|---|---|
| Argo CD | v3.5.5 | **v3.5.7** (dernier 3.5.x) |
| Chart `argo/argo-cd` | 10.10.3 (probable) | **10.11.7** (dernier chart en v3.5.x) |

- Le chart 10.10.5 reste en v3.5.5. Il faut donc passer à **10.11.7** pour obtenir la v3.5.7. Entre 10.10.3 et 10.11.7, le rendu avec `avp-values.yaml` ne diffère que par l'image `argocd` (v3.5.7, sidecars AVP compris), Dex v2.46.1 et les clés SSH de `ssh.bitbucket.org` ajoutées aux known hosts par défaut. Aucun CRD ni RBAC ne change, et `avp-values.yaml` ne demande **aucun ajustement**.
- **Limite** : v3.5.7 ne corrige **pas** les CVE remontées par Trivy. Le scan local de `quay.io/argoproj/argocd:v3.5.7` montre :
  - CVE-2025-68121 (Critical, Go stdlib) : toujours présente dans `kustomize` (go 1.24.0) et `git-lfs` (go 1.25.3) ;
  - CVE-2026-109518 (High, `apt` 3.2.0, Ubuntu 26.04) : toujours présente.

  La v3.6.1 (chart ≥ 10.12.1, dernier 10.15.0) supprime CVE-2025-68121, mais garde la CVE `apt`, pas encore corrigée dans l'image de base. Passer en 3.6 est un **changement de mineure** : lire d'abord [upgrading/3.5-3.6](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/3.5-3.6/).

## Procédure

```bash
# 0. Etat de depart : version du chart deploye et values effectives
helm list -n argocd                               # CHART = argo-cd-10.10.x ?
helm get values argocd -n argocd > /tmp/argocd-live-values.yaml
diff <(yq -P 'sort_keys(..)' /tmp/argocd-live-values.yaml) \
     <(yq -P 'sort_keys(..)' manuals/argocd/avp-values.yaml)   # doit etre vide, sinon reporter les ecarts dans avp-values.yaml AVANT

# 1. Chart
helm repo update argo
helm search repo argo/argo-cd --versions | head  # 10.11.7 -> v3.5.7

# 2. Diff (plugin helm-diff : helm plugin install https://github.com/databus23/helm-diff)
helm diff upgrade argocd argo/argo-cd -n argocd --version 10.11.7 \
  -f manuals/argocd/avp-values.yaml
#    Attendu : images argocd v3.5.5 -> v3.5.7 (dont sidecars avp-all/avp-helm),
#    dex v2.46.0 -> v2.46.1, known hosts bitbucket, labels de version. Rien d'autre.

# 3. Upgrade (sans --reuse-values : avp-values.yaml est la source de verite)
helm upgrade argocd argo/argo-cd -n argocd --version 10.11.7 \
  -f manuals/argocd/avp-values.yaml --wait --timeout 10m
```

## Vérifications

```bash
curl -s https://argocd.amonnier.fr/api/version          # {"Version":"v3.5.7"}
kubectl -n argocd get pods                              # tout Running, 0 restart recent
kubectl -n argocd get deploy argocd-repo-server \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}={.image}{"\n"}{end}'
#   argocd-repo-server, avp-all, avp-helm en v3.5.7 ; avp en ubi8
kubectl -n argocd logs deploy/argocd-repo-server -c avp-all --tail=20   # plugin enregistre, pas d'erreur
```

- UI ou CLI : toutes les Applications `Synced` / `Healthy`, aucune en `ComparisonError`.
- Plugins AVP : forcer un *Hard Refresh* sur `apps-inventory` (plugin `avpall`) et sur `bootstrap-secrets`. Les manifests doivent se générer et les `<path:...>` être substitués.
- Rollback : `helm rollback argocd <REVISION_PRECEDENTE> -n argocd` (`helm history argocd -n argocd`).

## Points de vigilance

- Lancer depuis un poste disposant de `kubectl` et `helm` (pas le VPS).
- `registry.access.redhat.com/ubi8` sans tag équivaut à `latest` : l'image du sidecar `avp` et de `download-tools` change à chaque nouveau pull, hors de tout contrôle de version.
- L'AVP téléchargé à chaque démarrage (`AVP_VERSION` 1.16.1) dépend de GitHub : sans réseau sortant, le repo-server ne démarre pas.
