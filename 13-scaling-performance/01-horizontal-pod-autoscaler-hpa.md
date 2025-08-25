🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1 Horizontal Pod Autoscaler (HPA)

## Qu'est-ce que l'Horizontal Pod Autoscaler ?

L'Horizontal Pod Autoscaler (HPA) est un contrôleur Kubernetes qui ajuste automatiquement le nombre de répliques d'un déploiement, ReplicaSet ou StatefulSet en fonction de métriques observées. Imaginez un restaurant qui ouvre plus de caisses quand la file d'attente s'allonge, puis les ferme quand le rush est passé. HPA fait exactement cela pour vos applications : il ajoute des pods quand la charge augmente et les retire quand elle diminue.

Dans un environnement MicroK8s de lab, HPA vous permet d'optimiser l'utilisation de vos ressources limitées en n'allouant que ce qui est nécessaire à chaque instant, tout en maintenant la performance de vos applications.

## Comment Fonctionne HPA ?

Le fonctionnement de HPA suit un cycle continu de surveillance et d'ajustement :

**Phase d'observation** : Toutes les 15 secondes par défaut, HPA interroge les métriques server pour obtenir les valeurs actuelles des métriques surveillées (CPU, mémoire, ou métriques custom).

**Phase de calcul** : HPA calcule le nombre de répliques nécessaire en utilisant la formule :
```
Répliques désirées = ceil(somme(métrique actuelle) / valeur cible)
```

**Phase de décision** : Si le nombre calculé diffère du nombre actuel de répliques et que le cooldown period est respecté (3 minutes pour scale up, 5 minutes pour scale down par défaut), HPA décide d'ajuster.

**Phase d'action** : HPA modifie le champ `replicas` de la ressource cible, déclenchant la création ou suppression de pods par le ReplicaSet controller.

## Prérequis pour Utiliser HPA

Avant de configurer HPA dans votre cluster MicroK8s, plusieurs éléments doivent être en place :

### Metrics Server

Le metrics server est indispensable pour HPA. Il collecte les métriques de ressources des nodes et pods. Pour l'activer dans MicroK8s :

```bash
microk8s enable metrics-server
```

Vérifiez que le metrics server fonctionne :
```bash
kubectl top nodes
kubectl top pods --all-namespaces
```

### Ressources Requests

Les pods que vous souhaitez autoscaler doivent définir des resource requests. Sans requests CPU, HPA ne peut pas calculer le pourcentage d'utilisation. Voici pourquoi c'est crucial :

```yaml
resources:
  requests:
    cpu: 100m      # Le pod demande 100 millicores
    memory: 128Mi  # Le pod demande 128 MiB de RAM
  limits:
    cpu: 500m      # Maximum 500 millicores
    memory: 512Mi  # Maximum 512 MiB
```

Sans ces requests, HPA ne sait pas si un pod utilisant 50m de CPU est à 50% ou à 5% de sa capacité cible.

## Configuration Basique de HPA

### Méthode 1 : Via kubectl (Approche Simple)

La façon la plus rapide de créer un HPA est d'utiliser la commande kubectl :

```bash
kubectl autoscale deployment mon-app --cpu-percent=50 --min=1 --max=10
```

Cette commande crée un HPA qui :
- Surveille le deployment "mon-app"
- Maintient l'utilisation CPU moyenne à 50%
- Garde entre 1 et 10 répliques

### Méthode 2 : Via Manifeste YAML (Approche Déclarative)

Pour plus de contrôle et de reproductibilité, utilisez un manifeste YAML :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mon-app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Ce manifeste configure un HPA plus sophistiqué qui surveille à la fois CPU et mémoire.

## Types de Métriques Supportées

HPA peut utiliser différents types de métriques pour prendre ses décisions :

### Métriques de Ressources

Les plus communes et simples à implémenter :

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization        # Pourcentage des requests
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      type: AverageValue       # Valeur absolue
      averageValue: "500Mi"    # Scale si > 500Mi par pod
```

### Métriques de Pods (Custom Metrics)

Pour des métriques exposées par les pods eux-mêmes :

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"      # 1000 requêtes/seconde par pod
```

### Métriques d'Objets

Basées sur des métriques d'autres objets Kubernetes :

```yaml
metrics:
- type: Object
  object:
    metric:
      name: requests-per-second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: main-route
    target:
      type: Value
      value: "10k"              # 10000 requêtes totales
```

### Métriques Externes

Provenant de systèmes externes au cluster :

```yaml
metrics:
- type: External
  external:
    metric:
      name: queue_messages
      selector:
        matchLabels:
          queue: "worker-tasks"
    target:
      type: AverageValue
      averageValue: "100"       # 100 messages par pod
```

## Comportement et Paramètres Avancés

### Stabilisation et Cooldown

HPA inclut des mécanismes pour éviter le "flapping" (oscillation rapide) :

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Fenêtre de 5 minutes
      policies:
      - type: Percent
        value: 50                      # Max 50% de pods en moins
        periodSeconds: 60              # Par minute
      - type: Pods
        value: 2                       # Max 2 pods en moins
        periodSeconds: 60              # Par minute
      selectPolicy: Min                # Politique la plus conservative
    scaleUp:
      stabilizationWindowSeconds: 60   # Fenêtre de 1 minute
      policies:
      - type: Percent
        value: 100                     # Double au maximum
        periodSeconds: 60
      - type: Pods
        value: 4                       # Max 4 pods ajoutés
        periodSeconds: 60
      selectPolicy: Max                # Politique la plus agressive
```

### Stratégies de Scaling

Vous pouvez définir comment HPA sélectionne les métriques quand plusieurs sont configurées :

```yaml
spec:
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # HPA utilisera la métrique qui demande le plus de répliques
```

## Surveillance et Debugging de HPA

### Vérifier le Status

Pour voir l'état actuel de vos HPA :

```bash
# Liste tous les HPA
kubectl get hpa

# Détails d'un HPA spécifique
kubectl describe hpa mon-app-hpa

# Observer en temps réel
kubectl get hpa mon-app-hpa --watch
```

### Interpréter les Outputs

Un output typique de `kubectl get hpa` :
```
NAME          REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
mon-app-hpa   Deployment/mon-app   45%/50%   1         10        4          10m
```

- **TARGETS** : Utilisation actuelle / cible
- **REPLICAS** : Nombre actuel de pods
- **45%/50%** : Utilisation CPU à 45%, cible à 50%

### Events et Logs

Pour comprendre les décisions de HPA :

```bash
# Events du HPA
kubectl describe hpa mon-app-hpa | grep -A 10 Events

# Logs du metrics server
kubectl logs -n kube-system deployment/metrics-server

# Events du deployment scalé
kubectl describe deployment mon-app | grep -A 10 Events
```

## Bonnes Pratiques pour HPA

### Dimensionnement des Requests

Définissez des requests réalistes basées sur des observations :

```yaml
# Mauvais : requests trop basses
resources:
  requests:
    cpu: 10m     # HPA va sur-réagir

# Bon : requests basées sur l'utilisation réelle
resources:
  requests:
    cpu: 100m    # Proche de l'utilisation normale
```

### Choix des Seuils

Les seuils dépendent de votre application :

- **Applications stateless** : 70-80% CPU est acceptable
- **Applications avec état** : 50-60% pour garder de la marge
- **APIs critiques** : 40-50% pour garantir la réactivité

### Limites Min/Max Appropriées

```yaml
# Pour un lab personnel
minReplicas: 1    # Économise les ressources au repos
maxReplicas: 5    # Évite de saturer le cluster

# Pour une app critique
minReplicas: 2    # Haute disponibilité
maxReplicas: 10   # Peut gérer les pics importants
```

### Combinaison de Métriques

Utilisez plusieurs métriques pour une décision plus intelligente :

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      type: Utilization
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 85
```

## Cas d'Usage Typiques dans un Lab

### Application Web Simple

Pour une application web dans votre lab :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

### Worker de Traitement

Pour un worker qui traite des tâches :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  minReplicas: 0    # Peut descendre à zéro
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metric:
        name: queue_length
      target:
        type: AverageValue
        averageValue: "10"  # 10 messages par worker
```

### API avec Patterns Prévisibles

Pour une API avec des pics connus :

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # Réaction rapide
      policies:
      - type: Percent
        value: 200                      # Double rapidement
        periodSeconds: 30
```

## Limitations et Considérations

### Limitations Techniques

HPA a certaines contraintes à connaître :

- **Délai de réaction** : Minimum 15 secondes entre les checks
- **Cooldown periods** : Empêche les ajustements trop fréquents
- **Métriques manquantes** : Si metrics-server est down, HPA ne scale pas
- **Pods en démarrage** : Les métriques des pods qui démarrent sont ignorées

### Considérations pour un Lab

Dans un environnement MicroK8s de lab :

- **Ressources limitées** : MaxReplicas doit tenir compte du total disponible
- **Coût du scaling** : Chaque pod consomme de la mémoire de base
- **Temps de démarrage** : Les images lourdes ralentissent le scaling
- **Stockage** : Les PVC peuvent limiter où les nouveaux pods peuvent s'exécuter

### Interactions avec d'Autres Fonctionnalités

HPA doit coexister avec :

- **VPA** : Ne pas utiliser les deux sur les mêmes métriques
- **Cluster Autoscaler** : Coordonner les limites
- **Pod Disruption Budgets** : Peuvent ralentir le scale down
- **Resource Quotas** : Peuvent empêcher le scale up

## Troubleshooting Commun

### HPA ne Scale pas

Vérifications à effectuer :

```bash
# Metrics server fonctionne ?
kubectl top nodes

# Requests définies ?
kubectl describe deployment mon-app | grep -A 5 Requests

# HPA voit les métriques ?
kubectl describe hpa mon-app-hpa | grep "Current CPU"

# Events d'erreur ?
kubectl get events --field-selector involvedObject.name=mon-app-hpa
```

### Scaling Trop Agressif

Si HPA scale trop rapidement :

- Augmentez stabilizationWindowSeconds
- Réduisez les pourcentages dans les policies
- Augmentez les seuils de déclenchement
- Vérifiez que les requests sont réalistes

### Métriques Incohérentes

Si les métriques semblent incorrectes :

```bash
# Comparer les métriques
kubectl top pods
kubectl get hpa

# Vérifier le calcul
kubectl get hpa mon-app-hpa -o yaml | grep -A 10 currentMetrics
```

## Monitoring et Observabilité

Pour surveiller efficacement vos HPA dans Grafana, surveillez ces métriques clés :

- `kube_horizontalpodautoscaler_status_current_replicas` : Nombre actuel de répliques
- `kube_horizontalpodautoscaler_status_desired_replicas` : Nombre désiré
- `kube_horizontalpodautoscaler_spec_max_replicas` : Maximum configuré
- `kube_horizontalpodautoscaler_spec_min_replicas` : Minimum configuré

Un dashboard efficace devrait montrer l'évolution du nombre de répliques en corrélation avec les métriques surveillées (CPU, mémoire, custom metrics) pour valider que HPA prend les bonnes décisions.

## Conclusion

L'Horizontal Pod Autoscaler est un outil puissant pour maintenir les performances tout en optimisant l'utilisation des ressources. Dans un environnement MicroK8s de lab, il permet d'expérimenter avec des stratégies de scaling sophistiquées tout en gérant efficacement les ressources limitées. La clé du succès avec HPA réside dans la compréhension de votre application, le choix de métriques pertinentes, et l'ajustement patient des paramètres basé sur l'observation du comportement réel.

⏭️
