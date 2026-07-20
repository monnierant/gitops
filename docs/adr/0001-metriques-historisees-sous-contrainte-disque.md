---
status: accepted
---

# Métriques historisées sous contrainte de disque

Le cluster `k8s1` dispose de **moins de 10 Gi d'espace libre** sur son nœud le plus confortable, partagés avec n8n, Obsidian et authentik — et avec `defaultReplicaCount: 1` sur Longhorn, un volume ne vit que sur un seul nœud, donc le budget réel n'est pas la somme des trois. On déploie malgré tout **kube-prometheus-stack**, mais délibérément bridé : scrape à 60 s, `retention: 7d` **et** `retentionSize: 2GB` sur un PVC de 3 Gi, soit environ 1,6 Gi de TSDB en régime établi contre ~6,8 Gi pour une configuration par défaut.

## Options considérées

- **VictoriaMetrics single-node** — environ 5x plus économe en disque à données égales (15 j tiendraient dans ~1,5 Gi). Écarté parce que le gain de disque ne compense pas la perte de l'écosystème clé en main : règles d'alerte à réécrire, pas de `ServiceMonitor` natif, et divergence avec les conventions de nommage que le dashboard TUI du projet attend déjà.
- **Remote-write vers Grafana Cloud** — zéro disque consommé sur le cluster. Écarté sur la volumétrie : le cluster produit environ 60–100 k séries actives pour un free tier plafonné à 10 k, ce qui imposerait un filtrage agressif, et ajouterait une dépendance WAN et un tiers externe pour des métriques d'homelab.
- **Prometheus seul, sans Grafana ni Alertmanager** — empreinte minimale, mais consultation en PromQL brut uniquement.

## Conséquences

- **`retentionSize` est le vrai garde-fou, pas `retention`.** La rétention en temps ne borne rien en cas de pic de cardinalité. Ne jamais retirer le cap en taille, même en augmentant la durée.
- **Le PVC est dimensionné à ~1,5x le `retentionSize`.** Le cap ne couvre ni le WAL ni les blocs en cours de compaction ; supprimer cette marge fait saturer le disque avant que le cap ne s'applique.
- **Pas de métriques du control-plane.** `kubeControllerManager`, `kubeScheduler` et `kubeEtcd` sont désactivés : sur Talos ces composants n'écoutent que sur `127.0.0.1`, et les scraper produirait trois cibles éternellement `DOWN` avec les alertes correspondantes en permanence au rouge. Les exposer suppose de modifier la machineconfig des 3 nœuds, donc de sortir du repo — même arbitrage que pour le TLS du kubelet côté metrics-server.
- **Le namespace `monitoring` est exempté de Pod Security Admission** (`enforce: privileged`, cf. [monitoring/namespace.yaml](../../monitoring/namespace.yaml)). Talos impose `baseline` par défaut, que node-exporter ne peut pas respecter : il lui faut `hostNetwork`, `hostPID`, `hostPort` et des `hostPath` sur `/proc`, `/sys` et `/`. Sans lui, plus de métriques d'usage disque des nœuds — soit précisément ce que ce stack existe pour surveiller. L'exemption profite aussi à Prometheus, Grafana et Alertmanager, qui n'en ont pas besoin ; l'isoler dans un namespace dédié coûtait une Application et un ServiceMonitor de plus.
- **Grafana est éphémère** (`persistence.enabled: false`). Les dashboards livrés par le chart viennent de ConfigMaps, mais **tout dashboard créé à la main dans l'UI sera perdu au redémarrage du pod**. Les dashboards durables doivent être versionnés dans ce repo.
- **Le mot de passe admin Grafana passe par ESO, pas par AVP.** Le chart accepte un `admin.existingSecret`, donc un `ExternalSecret` suffit — même pattern que `authentik-creds`. C'est pourquoi le manifeste vit dans `infra/` et non `infra-with-secrets/` : aucun `<path:...>` à rendre. Règle générale : AVP n'est nécessaire que lorsqu'une valeur doit être injectée dans le manifeste au moment du rendu.
