# GitOps k8s1

Repo GitOps qui bootstrape et pilote le cluster Talos `k8s1` via Argo CD. Ce fichier est un glossaire : il fixe le vocabulaire du projet, pas ses choix d'implémentation (voir [README.md](README.md) pour ceux-là).

## Language

### Observabilité

**Métriques instantanées**:
Valeurs de consommation CPU/mémoire lues à l'instant T, non persistées. Servent à `kubectl top`, aux HPA et aux dashboards temps réel.
_Avoid_: monitoring, metrics-server (c'est l'implémentation, pas le concept)

**Métriques historisées**:
Séries temporelles conservées sur une fenêtre de rétention bornée, permettant de regarder en arrière. Consomment du disque proportionnellement à la rétention.
_Avoid_: monitoring, métriques, Prometheus

**Logs**:
Sorties texte des conteneurs, indexées et interrogeables. Distinctes des métriques historisées et nettement plus coûteuses en stockage.
_Avoid_: monitoring, observabilité

**Rétention**:
Durée au-delà de laquelle une donnée d'observabilité est supprimée. C'est le levier principal de maîtrise du disque sur ce cluster.

> **Note** : « monitoring » seul est proscrit dans ce repo. Le terme recouvre trois concepts aux coûts de stockage radicalement différents ; toujours préciser lequel.
