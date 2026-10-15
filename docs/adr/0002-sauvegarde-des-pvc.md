---
status: proposed
---

# Sauvegarde des PVC

Aujourd'hui, **aucun volume du cluster `k8s1` n'est sauvegardé** : Longhorn tourne avec `defaultReplicaCount: 1`, sans backup target, sans RecurringJob, et aucun outil de sauvegarde n'est déployé. Or plusieurs applications gardent dans leur PVC une configuration fabriquée depuis l'UI et absente de Git : Sonarr, Radarr, Bazarr, Prowlarr, Seerr et n8n (SQLite), ainsi qu'Authentik (Postgres). Il faut y ajouter des données métier : naokicard (SQLite), Postiz (Postgres et uploads) et Obsidian. L'incident tal02 du 2026-07-07 a montré qu'un seul nœud perdu suffit à rendre un volume `faulted`. Si le disque avait lâché, la base de Sonarr était perdue.

On met en place les **sauvegardes natives Longhorn vers un partage NFS dédié du NAS**, déclenchées par des RecurringJobs en opt-in. Les restaurations passent par des **snapshots CSI déclaratifs**. La conversion de nos StatefulSets en Deployments sert de premier exercice de restauration grandeur nature.

## Décisions

- **Périmètre de la menace, en deux phases.** La phase 1 protège contre la perte d'un nœud ou d'un disque et contre une erreur humaine, avec une copie sur le NAS du LAN. La phase 2 protège contre la perte du NAS ou du site (incendie, vol, ransomware), avec une copie hors site. Les choix de la phase 1 ne doivent pas bloquer la phase 2.
- **Cible : un partage NFS dédié** (`/Volume1/Backups`, sous-dossier `Longhorn`, sur `192.168.1.19`, soit la cible `nfs://192.168.1.19:/Volume1/Backups/Longhorn`), distinct de `MediaCenter`. Les applications média, dont Bazarr compromis le 2026-10-13, ne doivent pas pouvoir écrire dans les sauvegardes. L'export porte `insecure`, faute de quoi Twingate casse le montage en EPERM. Le NAS ne voit jamais l'IP des nœuds (`10.0.0.x`), seulement celle du connecteur Twingate, `192.168.1.17`, qui fait du NAT. L'export est donc limité à cette IP, réservée en DHCP. Le vrai contrôle d'accès réside dans la ressource Twingate, que seul le service account du cluster doit pouvoir atteindre.
- **Opt-in par label, sans groupe `default`.** Un volume entre dans les sauvegardes quand sa PVC porte `recurring-job-group.longhorn.io/critique: enabled` et `recurring-job.longhorn.io/source: enabled`, ce qui rend le périmètre lisible dans Git. Le groupe `default` de Longhorn, qui s'applique implicitement à tout volume sans label, n'est pas utilisé.
- **Politique du groupe `critique` :** un snapshot local toutes les 6 h (4 conservés), un backup quotidien à 03:00 (7 conservés) et un backup hebdomadaire (4 conservés). Le RPO est de 24 h pour une perte de disque et de 6 h pour une erreur dans l'UI, avec un mois de profondeur pour remonter avant une compromission.
- **Volumes `critique` :** les PVC de config de Sonarr, Radarr, Bazarr et Prowlarr, celles de Seerr, n8n et Obsidian, `naokicard-prod-data`, le Postgres et les uploads de Postiz, et les Postgres de Keycloak et d'Authentik. **Exclus :** Prometheus, Redis, le staging, les PVC `-data` des *arr (du cache) et les PVC orphelines.
- **Restauration déclarative par snapshots CSI.** On installe `snapshot-controller` avec ses CRDs, et une `VolumeSnapshotClass` Longhorn de `type: bak`. Une restauration s'écrit dans Git : une `VolumeSnapshotContent` pointant vers le backup, une `VolumeSnapshot`, puis une PVC avec `dataSource`. Ce sont les mêmes briques que le plugin CSI de Velero utilisera en phase 2.
- **On reste à 1 replica.** C'est un choix assumé, qui ne tient pas au disque : au 2026-10-15, Longhorn dispose d'environ 500 G libres, dont 228 G sur `k8s1-tal06-w`. La contrainte de moins de 10 Gi notée dans l'[ADR 0001](0001-metriques-historisees-sous-contrainte-disque.md) date d'avant l'ajout des workers. Le backup protège contre la perte de données, pas contre l'indisponibilité. Une panne de nœud qui touche un volume critique se règle par une restauration.
- **Les Postgres restent crash-consistent en phase 1.** Un snapshot Longhorn est atomique sur le volume, et les données comme le WAL vivent sur le même volume : Postgres repart en rejouant le WAL. On le prouve par un exercice de restauration sur le Postgres de Keycloak. Les dumps logiques (`pg_dump`) arrivent en phase 2, sous forme de hooks Velero.
- **Supervision.** Une `PrometheusRule` lève deux alertes, envoyées sur Discord : aucun backup réussi depuis 26 h (`longhorn_volume_last_backup_at`), et un backup en `Error`. Longhorn n'expose aucune métrique de disponibilité du backup target : un NAS injoignable se voit d'abord par des backups en erreur. On y ajoute le `ServiceMonitor` Longhorn et un dashboard « Longhorn - Sauvegardes » dans le dossier Grafana Securite.
- **Exercices de restauration trimestriels** guidés par un runbook. On restaure un volume critique tiré au hasard dans un namespace jetable, on vérifie que l'application démarre, puis on supprime le namespace. Chaque exercice est noté dans un journal.

## Migration StatefulSet → Deployment

Les StatefulSets de `personal-rr` (Sonarr, Radarr, Bazarr, Prowlarr, Seerr, lighthouse), ceux de n8n et celui d'Obsidian n'ont aucune raison d'en être : ils tournent à un seul replica et n'ont besoin ni d'identité stable ni d'ordre de démarrage. Les Postgres et Redis restent en StatefulSet.

Sept d'entre eux montent déjà une PVC autonome via `claimName`. Seul lighthouse utilise un `volumeClaimTemplates`. Changer le `kind` suffirait donc. On choisit pourtant de **migrer par restauration**, pour que chaque application serve de test de bout en bout :

1. Lancer un backup Longhorn de la PVC actuelle.
2. Déclarer dans Git la `VolumeSnapshotContent`, la `VolumeSnapshot` et la nouvelle PVC restaurée (par ex. `prowlarr-config`).
3. Remplacer le StatefulSet par un Deployment en **`strategy: Recreate`**, qui monte la nouvelle PVC. Avec un RollingUpdate, l'attache RWO se bloquerait en Multi-Attach.
4. Vérifier dans l'UI que la configuration est intacte.
5. Lancer un backup final de l'ancienne PVC, puis la supprimer avec son PV et son volume Longhorn. Le `reclaimPolicy: Retain` impose de nettoyer le PV et le volume à la main.

**Prowlarr sert de pilote** : il a peu de configuration et se reconfigure facilement au pire. Le runbook s'écrit pendant ce pilote. Suivent le reste des *arr, Seerr et lighthouse, puis n8n et Obsidian. Les orphelines `overseer-data` et `lighthouse-data` reçoivent le même traitement : un backup final, puis la suppression.

## Ordre de mise en œuvre

1. Créer l'export NFS sur le NAS (opération manuelle, hors repo).
2. Infra : backup target Longhorn, RecurringJobs, `snapshot-controller`, `VolumeSnapshotClass`, alertes et dashboard.
3. Labels `critique` sur les PVC, dans ce repo et dans `private-gitops`. Attendre un premier backup réussi pour chaque volume.
4. Pilote Prowlarr et rédaction du runbook.
5. Migration des autres StatefulSets.
6. Exercice de restauration sur le Postgres de Keycloak.

## Options considérées

- **Velero + Kopia dès la phase 1.** C'est le standard du marché : il sauvegarde aussi les objets Kubernetes et gère des hooks applicatifs. Écarté pour la phase 1, car il exige un S3 (un MinIO à monter sur le NAS) et plus de pièces à faire tourner, alors que rien n'est sauvegardé aujourd'hui. Il est retenu pour la phase 2, vers un S3 hors site.
- **VolSync (restic).** Restauration continue possible, mais une paire de CR par PVC et un opérateur de plus. Écarté.
- **Sous-dossier de `MediaCenter` comme cible.** Rien à créer, mais les sauvegardes partageraient les droits des applications média. Écarté.
- **Restauration par l'UI Longhorn puis PV statique**, sur le modèle de `apps/n8n/pv.yaml`. Gardé comme repli si la voie CSI bloque pendant le pilote.
- **2 replicas via une StorageClass `longhorn-critique`.** Les PVC restaurées l'auraient prise pendant la migration, pour un surcoût d'environ 40 G. Cette option aurait évité de restaurer à chaque panne de nœud. Écartée pour rester à 1 replica.
- **Suppression différée des anciennes PVC (14 jours).** Écartée au profit d'une suppression dès la validation.

## Conséquences

- **La moitié des volumes vit sur un seul nœud.** Au 2026-10-15, `k8s1-tal06-w` porte 22 des 44 replicas. À 1 replica, sa perte met environ la moitié des volumes en `faulted` en même temps. Le runbook doit donc prévoir une restauration en série, et pas seulement celle d'un volume isolé.
- **Les snapshots locaux consomment du disque sur le nœud** qui porte le replica. C'est sans enjeu avec l'espace actuel, mais à surveiller sur `k8s1-tal04-w` et `k8s1-tal05-w`, qui ont 14 G libres chacun.
- **Pendant une migration, un volume existe en double.** Le nouveau volume restauré cohabite avec l'ancien jusqu'à la validation. On migre une application à la fois.
- **Deux nœuds Longhorn fantômes** (`talos-7bm-xuv` et `talos-qsa-2dd`, `Ready=False`, sans replica) restent à supprimer de Longhorn. C'est du ménage, indépendant de cette décision.
- **Après la validation, le filet de secours n'est que le backup final de l'ancienne PVC.** Il disparaît avec la rétention, donc au bout d'environ 4 semaines au plus. Un défaut découvert plus tard ne se rattrape plus.
- **Le RTO dépend de la restauration.** À 1 replica, la perte d'un nœud se paie par une restauration manuelle. Le runbook doit pouvoir s'appliquer sans réfléchir.
- **La phase 1 ne protège pas contre la perte du NAS.** Les sauvegardes et les médias vivent dans le même boîtier, et ce reste à couvrir est l'objet de la phase 2.
