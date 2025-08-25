🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2 Vertical Pod Autoscaler (VPA)

## Qu'est-ce que le Vertical Pod Autoscaler ?

Le Vertical Pod Autoscaler (VPA) est un composant Kubernetes qui ajuste automatiquement les requests et limits de CPU et mémoire des containers. Contrairement à HPA qui ajoute ou retire des pods, VPA modifie la taille des pods existants. Imaginez que vous ayez une voiture : HPA serait comme ajouter plus de voitures pour transporter plus de personnes, tandis que VPA serait comme échanger votre voiture contre un bus quand vous avez besoin de plus de capacité.

Dans un lab MicroK8s où les ressources sont précieuses, VPA vous aide à trouver la taille optimale pour chaque pod, évitant le gaspillage tout en garantissant que vos applications ont suffisamment de ressources pour fonctionner correctement.

## Pourquoi VPA est Important dans un Lab

### Le Problème du Dimensionnement

Quand vous déployez une application, vous devez spécifier les ressources :

```yaml
resources:
  requests:
    cpu: 100m      # Est-ce trop ? Pas assez ?
    memory: 128Mi  # Comment savoir ?
  limits:
    cpu: 500m      # Quelle marge prévoir ?
    memory: 512Mi  # Et si l'app évolue ?
```

La plupart des développeurs estiment ces valeurs, souvent incorrectement. Résultat :
- **Sur-allocation** : Pods réservant des ressources inutilisées, empêchant d'autres pods de s'exécuter
- **Sous-allocation** : Pods constamment throttled (CPU) ou killed (OOM - Out Of Memory)

VPA résout ce problème en observant l'utilisation réelle et en ajustant automatiquement.

### Bénéfices Spécifiques pour un Lab Personnel

Dans votre environnement MicroK8s limité :
- **Optimisation automatique** : Plus besoin de deviner les bonnes valeurs
- **Adaptation dynamique** : Les ressources s'ajustent quand le comportement de l'app change
- **Maximisation de la densité** : Plus d'applications peuvent coexister sur le même hardware
- **Apprentissage continu** : VPA apprend les patterns d'utilisation au fil du temps

## Comment Fonctionne VPA

VPA opère selon trois composants principaux :

### 1. Recommender

Le cerveau de VPA qui :
- Collecte l'historique d'utilisation des ressources via Metrics Server
- Analyse les patterns sur plusieurs jours
- Calcule les recommandations optimales
- Utilise des percentiles pour éviter les outliers

### 2. Updater

Le composant d'action qui :
- Compare les recommandations aux valeurs actuelles
- Décide si une mise à jour est nécessaire
- Marque les pods pour éviction si les changements sont significatifs
- Respecte les PodDisruptionBudgets

### 3. Admission Controller

Le gardien qui :
- Intercepte les créations de pods
- Applique les recommandations VPA aux nouveaux pods
- Garantit que les pods démarrent avec les bonnes ressources
- Valide que les valeurs sont dans les limites acceptables

### Cycle de Vie d'un Ajustement

```
1. Pod démarre avec resources initiales
2. VPA observe l'utilisation (CPU, mémoire)
3. Recommender calcule les valeurs optimales
4. Si écart significatif → Updater évince le pod
5. Admission Controller applique les nouvelles valeurs
6. Pod redémarre avec resources optimisées
```

## Installation de VPA dans MicroK8s

VPA n'est pas inclus par défaut dans MicroK8s. Voici comment l'installer :

### Étape 1 : Cloner le Repository

```bash
# Cloner le repo officiel
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
```

### Étape 2 : Déployer VPA

```bash
# Installer les CRDs et composants
./hack/vpa-up.sh
```

### Étape 3 : Vérifier l'Installation

```bash
# Vérifier que les pods VPA sont running
kubectl get pods -n kube-system | grep vpa

# Devrait montrer :
# vpa-admission-controller-xxx   1/1     Running
# vpa-recommender-xxx            1/1     Running
# vpa-updater-xxx                1/1     Running
```

### Configuration Alternative pour MicroK8s

Si vous préférez une installation manuelle contrôlée :

```yaml
# vpa-crd.yaml - Custom Resource Definitions
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: verticalpodautoscalers.autoscaling.k8s.io
spec:
  group: autoscaling.k8s.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
  scope: Namespaced
  names:
    plural: verticalpodautoscalers
    singular: verticalpodautoscaler
    kind: VerticalPodAutoscaler
    shortNames:
    - vpa
```

## Configuration de Base de VPA

### Mode "Auto" - Ajustement Automatique

Le mode le plus simple où VPA gère tout :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Auto"  # VPA ajuste automatiquement
```

### Mode "Off" - Recommandations Seulement

Pour observer sans modifier :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Off"  # Recommandations seulement
```

### Mode "Initial" - Configuration au Démarrage

Applique les recommandations uniquement aux nouveaux pods :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Initial"  # Seulement pour les nouveaux pods
```

### Mode "Recreate" - Recréation Active

Force la recréation des pods pour appliquer les changements :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Recreate"  # Recrée les pods si nécessaire
```

## Configuration Avancée

### Contraintes de Ressources

Définir des limites pour VPA :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mon-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: mon-container
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

### Politique par Container

Pour les pods multi-containers :

```yaml
spec:
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1
        memory: 1Gi
      mode: "Auto"
    - containerName: sidecar
      minAllowed:
        cpu: 10m
        memory: 32Mi
      maxAllowed:
        cpu: 100m
        memory: 128Mi
      mode: "Off"  # Pas d'auto-scaling pour le sidecar
```

### Contrôle des Évictions

Gérer quand et comment les pods sont recréés :

```yaml
spec:
  updatePolicy:
    updateMode: "Auto"
    evictionRequirements:
      # Ne pas évincer si changement < 10%
      changeRequirement: 0.1
      # Attendre au moins 12h entre les évictions
      minReplicas: 1
```

## Recommandations et Métriques

### Comprendre les Recommandations

VPA génère quatre types de recommandations :

```yaml
status:
  recommendation:
    containerRecommendations:
    - containerName: mon-container
      # Valeur minimale absolue (percentile 0)
      lowerBound:
        cpu: 25m
        memory: 64Mi
      # Valeur recommandée (percentile 90)
      target:
        cpu: 100m
        memory: 256Mi
      # Valeur recommandée sans contraintes
      uncappedTarget:
        cpu: 150m
        memory: 384Mi
      # Valeur maximale observée (percentile 95)
      upperBound:
        cpu: 500m
        memory: 1Gi
```

### Interpréter les Valeurs

- **LowerBound** : Minimum absolu, en dessous l'app ne fonctionne pas
- **Target** : Valeur optimale pour la plupart des cas
- **UncappedTarget** : Ce que VPA recommanderait sans vos contraintes
- **UpperBound** : Pour gérer les pics, rarement dépassé

### Consulter les Recommandations

```bash
# Voir toutes les VPA
kubectl get vpa

# Détails d'une VPA spécifique
kubectl describe vpa mon-app-vpa

# Recommandations en format JSON
kubectl get vpa mon-app-vpa -o json | jq '.status.recommendation'

# Historique des recommandations
kubectl get vpa mon-app-vpa -o json | jq '.status.conditions'
```

## Stratégies d'Utilisation dans un Lab

### Stratégie 1 : Discovery Mode

Commencez en mode "Off" pour apprendre :

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: discovery-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Off"
```

Après quelques jours, examinez les recommandations et ajustez manuellement vos déploiements.

### Stratégie 2 : Progressive Adoption

Adoptez VPA progressivement :

1. **Phase 1** : Mode "Off" pour observer (1 semaine)
2. **Phase 2** : Mode "Initial" pour les nouveaux pods (1 semaine)
3. **Phase 3** : Mode "Auto" avec contraintes strictes
4. **Phase 4** : Relâcher progressivement les contraintes

### Stratégie 3 : Environnements Différenciés

Utilisez différents modes selon l'environnement :

```yaml
# Développement : Auto avec larges limites
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: dev-vpa
spec:
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      maxAllowed:
        cpu: 4
        memory: 8Gi

---
# Production : Initial avec limites strictes
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: prod-vpa
spec:
  updatePolicy:
    updateMode: "Initial"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      maxAllowed:
        cpu: 1
        memory: 2Gi
```

## VPA vs HPA : Comparaison et Combinaison

### Différences Fondamentales

| Aspect | HPA | VPA |
|--------|-----|-----|
| **Action** | Change le nombre de pods | Change la taille des pods |
| **Métrique principale** | Utilisation moyenne | Historique d'utilisation |
| **Temps de réaction** | Secondes à minutes | Heures à jours |
| **Disruption** | Ajoute/retire des pods | Recrée les pods existants |
| **Use case** | Charge variable | Besoins stables mais mal estimés |

### Quand Utiliser Quoi

**Utilisez HPA quand** :
- La charge varie significativement dans le temps
- L'application est stateless
- Vous pouvez paralléliser le travail
- Les temps de démarrage sont courts

**Utilisez VPA quand** :
- La charge est relativement stable
- Vous ne connaissez pas les bonnes valeurs de ressources
- L'application est stateful
- Vous voulez optimiser les coûts/ressources

### Combinaison HPA + VPA

Il est possible mais délicat de combiner les deux :

```yaml
# VPA pour optimiser les ressources
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: combo-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledResources: ["memory"]  # VPA gère seulement la mémoire

---
# HPA pour gérer la charge
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: combo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mon-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu  # HPA gère seulement le CPU
      target:
        type: Utilization
        averageUtilization: 70
```

**Important** : Ne jamais laisser HPA et VPA gérer la même métrique !

## Limitations et Considérations

### Limitations Techniques

VPA a plusieurs contraintes importantes :

- **Éviction de pods** : VPA doit recréer les pods pour changer les ressources
- **Temps d'apprentissage** : Nécessite plusieurs jours de données
- **Pas de support StatefulSet complet** : Limité pour les workloads stateful
- **Conflit avec HPA** : Complexe à combiner sur les mêmes métriques

### Considérations pour un Lab MicroK8s

Dans votre environnement de lab :

- **Ressources totales limitées** : VPA peut recommander plus que disponible
- **Redémarrages fréquents** : Peuvent perturber vos tests
- **Données insuffisantes** : Les apps de test courte durée n'ont pas assez d'historique
- **Overhead des petits pods** : VPA peut sur-optimiser et créer des pods trop petits

### Impact sur la Stabilité

VPA peut affecter la stabilité :

```yaml
# Configuration conservative pour la stabilité
spec:
  updatePolicy:
    updateMode: "Initial"  # Évite les redémarrages
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        memory: 100Mi  # Évite les pods trop petits
      maxAllowed:
        memory: 1Gi   # Évite l'épuisement des ressources
```

## Monitoring et Observabilité

### Métriques Prometheus pour VPA

Surveillez ces métriques clés :

```yaml
# Recommandations VPA
vpa_recommender_recommendation{container="mon-container",resource="cpu"}
vpa_recommender_recommendation{container="mon-container",resource="memory"}

# Évictions VPA
vpa_updater_evictions_total

# Âge des recommandations
vpa_recommender_last_sample_age_seconds
```

### Dashboard Grafana pour VPA

Un bon dashboard VPA devrait montrer :

1. **Graphique temporel** : Évolution des recommandations vs utilisation réelle
2. **Tableau de bord** : État actuel de toutes les VPA
3. **Alertes** : Recommandations drastiquement différentes des valeurs actuelles
4. **Historique** : Nombre d'évictions et leur impact

### Requêtes PromQL Utiles

```promql
# Écart entre recommandation et actuel
abs(vpa_recommender_recommendation - container_spec_cpu_quota)
  / container_spec_cpu_quota > 0.5

# Pods en attente d'éviction
count(vpa_updater_pods_eviction_queue_length > 0)

# Efficacité de VPA (utilisation vs requests)
container_cpu_usage_seconds_total
  / container_spec_cpu_shares
```

## Troubleshooting Commun

### VPA ne Fait pas de Recommandations

Vérifications :

```bash
# VPA est-il actif ?
kubectl get vpa mon-app-vpa

# Y a-t-il des erreurs ?
kubectl describe vpa mon-app-vpa | grep -A 10 Conditions

# Metrics Server fonctionne ?
kubectl top pods

# Historique suffisant ?
kubectl get vpa mon-app-vpa -o json | jq '.status.recommendation'
```

### Recommandations Irréalistes

Si VPA recommande des valeurs étranges :

```bash
# Vérifier les percentiles utilisés
kubectl get vpa mon-app-vpa -o yaml | grep -A 20 recommendation

# Examiner l'historique
kubectl logs -n kube-system deployment/vpa-recommender

# Ajuster les contraintes
kubectl edit vpa mon-app-vpa
```

### Pods Constamment Évincés

Pour réduire les évictions :

```yaml
spec:
  updatePolicy:
    updateMode: "Initial"  # Ou passer en mode "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m      # Augmenter les minimums
        memory: 256Mi
```

### Conflits avec Autres Ressources

Résoudre les conflits :

```bash
# Vérifier les Resource Quotas
kubectl get resourcequota

# Vérifier les LimitRanges
kubectl get limitrange

# Vérifier les PodDisruptionBudgets
kubectl get pdb
```

## Bonnes Pratiques

### 1. Commencer Prudemment

- Toujours débuter en mode "Off"
- Observer pendant au moins 7 jours
- Valider les recommandations avant d'automatiser

### 2. Définir des Garde-fous

```yaml
resourcePolicy:
  containerPolicies:
  - containerName: "*"
    minAllowed:
      cpu: 10m       # Jamais en dessous
      memory: 64Mi
    maxAllowed:
      cpu: 2         # Jamais au dessus
      memory: 4Gi
```

### 3. Monitorer l'Impact

- Suivre les métriques avant/après VPA
- Mesurer l'impact sur la performance
- Documenter les changements

### 4. Adapter au Contexte

- **Apps critiques** : Mode "Initial" ou "Off"
- **Dev/Test** : Mode "Auto" agressif
- **Batch jobs** : VPA peut être très efficace
- **Services temps réel** : Attention aux évictions

### 5. Réviser Régulièrement

- Examiner les recommandations mensuellement
- Ajuster les contraintes selon l'évolution
- Nettoyer les VPA obsolètes

## Cas d'Usage Avancés

### Multi-Container avec Priorités

```yaml
spec:
  resourcePolicy:
    containerPolicies:
    # Container principal : auto-scaling complet
    - containerName: main-app
      minAllowed:
        cpu: 100m
        memory: 256Mi
      maxAllowed:
        cpu: 2
        memory: 4Gi
      controlledValues: RequestsAndLimits

    # Sidecar logging : ressources fixes
    - containerName: fluentd
      mode: "Off"

    # Proxy : seulement la mémoire
    - containerName: envoy
      controlledResources: ["memory"]
      minAllowed:
        memory: 128Mi
      maxAllowed:
        memory: 512Mi
```

### VPA pour Jobs et CronJobs

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: batch-job-vpa
spec:
  targetRef:
    apiVersion: batch/v1
    kind: Job
    name: data-processing
  updatePolicy:
    updateMode: "Initial"  # Applique aux nouvelles exécutions
  resourcePolicy:
    containerPolicies:
    - containerName: processor
      minAllowed:
        memory: 512Mi  # Les jobs ont souvent besoin de mémoire
      maxAllowed:
        memory: 16Gi
```

## Conclusion

Le Vertical Pod Autoscaler est un outil puissant pour optimiser automatiquement l'utilisation des ressources dans votre cluster MicroK8s. Bien qu'il nécessite une période d'apprentissage et une configuration soignée, VPA peut considérablement améliorer l'efficacité de votre lab en trouvant le juste équilibre entre performance et consommation de ressources. La clé du succès réside dans une adoption progressive, un monitoring attentif, et une compréhension claire de vos workloads. Dans un environnement de lab aux ressources limitées, VPA devient un allié précieux pour maximiser ce que vous pouvez faire avec le hardware disponible.

⏭️
