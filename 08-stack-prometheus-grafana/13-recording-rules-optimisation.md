🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.13 Recording Rules et Optimisation

## Introduction aux Recording Rules

Les recording rules sont une fonctionnalité puissante de Prometheus qui permet de pré-calculer des requêtes complexes et de stocker leurs résultats sous forme de nouvelles séries temporelles. Cette approche améliore considérablement les performances de votre système de monitoring, particulièrement important dans un environnement de lab personnel où les ressources peuvent être limitées.

Imaginez que vous ayez une requête PromQL complexe que vous utilisez fréquemment dans vos dashboards Grafana. Sans recording rules, cette requête serait exécutée à chaque rafraîchissement du dashboard, consommant des ressources CPU et mémoire. Avec les recording rules, le calcul est fait une fois par Prometheus et le résultat est stocké, prêt à être consulté instantanément.

## Pourquoi utiliser des Recording Rules ?

### Performance et rapidité
Lorsque vous consultez un dashboard Grafana avec plusieurs panels complexes, chaque panel exécute ses requêtes PromQL contre Prometheus. Si ces requêtes impliquent des agrégations sur de longues périodes ou de nombreuses séries temporelles, le temps de réponse peut devenir problématique. Les recording rules transforment ces calculs coûteux en simples lectures de métriques pré-calculées.

### Réduction de la charge système
Dans un lab MicroK8s personnel, vos ressources sont précieuses. Les recording rules permettent de :
- Diminuer la charge CPU lors des consultations de dashboards
- Réduire l'utilisation mémoire pendant les requêtes
- Améliorer la réactivité globale de votre stack de monitoring

### Standardisation des métriques
Les recording rules vous permettent de créer des métriques standardisées que toute votre équipe (ou vous-même dans différents contextes) peut utiliser de manière cohérente. Au lieu de recréer la même logique complexe dans plusieurs dashboards, vous définissez la règle une fois.

## Anatomie d'une Recording Rule

Une recording rule se compose de plusieurs éléments essentiels :

```yaml
groups:
  - name: example_group
    interval: 30s
    rules:
      - record: job:request_rate:5m
        expr: rate(http_requests_total[5m])
        labels:
          environment: lab
```

Décortiquons cette structure :
- **groups** : Les règles sont organisées en groupes pour une meilleure gestion
- **name** : Identifiant unique du groupe de règles
- **interval** : Fréquence d'évaluation des règles (par défaut, celle de Prometheus)
- **record** : Nom de la nouvelle métrique créée
- **expr** : Expression PromQL à évaluer
- **labels** : Labels additionnels à ajouter à la métrique résultante

## Convention de nommage

Prometheus recommande une convention de nommage spécifique pour les recording rules qui facilite leur identification et leur utilisation :

```
level:metric:operations
```

Par exemple :
- `instance:node_cpu:rate5m` - Taux CPU par instance sur 5 minutes
- `job:request_duration:mean5m` - Durée moyenne des requêtes par job sur 5 minutes
- `cluster:node_cpu:sum` - Somme totale du CPU pour le cluster

Cette convention aide à comprendre immédiatement :
- Le niveau d'agrégation (instance, job, cluster)
- La métrique source (node_cpu, request_duration)
- L'opération appliquée (rate5m, mean5m, sum)

## Création de Recording Rules pour MicroK8s

### Fichier de configuration

Dans MicroK8s, les recording rules sont généralement stockées dans un ConfigMap Kubernetes. Créez d'abord votre fichier de règles :

```yaml
# recording-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-recording-rules
  namespace: observability
data:
  recording-rules.yml: |
    groups:
      # Règles pour les métriques de performance des pods
      - name: pod_performance
        interval: 30s
        rules:
          # Utilisation CPU moyenne par pod sur 5 minutes
          - record: pod:container_cpu_usage:rate5m
            expr: |
              sum by (pod, namespace) (
                rate(container_cpu_usage_seconds_total[5m])
              )

          # Utilisation mémoire par pod
          - record: pod:container_memory_usage:current
            expr: |
              sum by (pod, namespace) (
                container_memory_working_set_bytes
              )

      # Règles pour les métriques de cluster
      - name: cluster_metrics
        interval: 60s
        rules:
          # Utilisation CPU totale du cluster
          - record: cluster:cpu_usage:rate5m
            expr: |
              sum(
                pod:container_cpu_usage:rate5m
              )

          # Pourcentage d'utilisation CPU du cluster
          - record: cluster:cpu_usage_percentage:rate5m
            expr: |
              cluster:cpu_usage:rate5m
              /
              sum(machine_cpu_cores)
              * 100
```

### Application des règles

Pour appliquer ces règles dans votre cluster MicroK8s :

```bash
kubectl apply -f recording-rules.yaml
```

Ensuite, vous devez configurer Prometheus pour charger ces règles. Modifiez la configuration Prometheus :

```yaml
# Dans la ConfigMap de Prometheus
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

# Le reste de votre configuration...
```

## Recording Rules avancées pour l'optimisation

### Agrégations multi-niveaux

Pour optimiser davantage, créez des règles qui s'appuient sur d'autres règles :

```yaml
groups:
  - name: hierarchical_rules
    rules:
      # Niveau 1 : Calcul de base
      - record: instance:node_network_receive_bytes:rate5m
        expr: rate(node_network_receive_bytes_total[5m])

      # Niveau 2 : Agrégation par job
      - record: job:node_network_receive_bytes:rate5m
        expr: |
          sum by (job) (
            instance:node_network_receive_bytes:rate5m
          )

      # Niveau 3 : Total cluster
      - record: cluster:node_network_receive_bytes:rate5m
        expr: sum(job:node_network_receive_bytes:rate5m)
```

### Règles pour les percentiles

Les calculs de percentiles sont particulièrement coûteux. Pré-calculez-les :

```yaml
- name: latency_percentiles
  interval: 60s
  rules:
    - record: job:request_latency:p50_5m
      expr: |
        histogram_quantile(0.5,
          sum by (job, le) (
            rate(http_request_duration_seconds_bucket[5m])
          )
        )

    - record: job:request_latency:p95_5m
      expr: |
        histogram_quantile(0.95,
          sum by (job, le) (
            rate(http_request_duration_seconds_bucket[5m])
          )
        )

    - record: job:request_latency:p99_5m
      expr: |
        histogram_quantile(0.99,
          sum by (job, le) (
            rate(http_request_duration_seconds_bucket[5m])
          )
        )
```

## Optimisation de la rétention et du stockage

### Configuration de la rétention différenciée

Les recording rules permettent d'optimiser le stockage en conservant des données agrégées plus longtemps que les métriques brutes :

```yaml
# Règles pour données long terme
- name: long_term_aggregations
  interval: 5m
  rules:
    # Moyenne horaire pour conservation longue durée
    - record: job:cpu_usage:avg1h
      expr: |
        avg_over_time(
          job:container_cpu_usage:rate5m[1h]
        )

    # Moyenne journalière
    - record: job:cpu_usage:avg1d
      expr: |
        avg_over_time(
          job:cpu_usage:avg1h[24h]
        )
```

### Downsampling intelligent

Créez des métriques à différentes résolutions temporelles :

```yaml
- name: downsampling_rules
  rules:
    # Résolution fine (1 minute)
    - record: service:requests:rate1m
      expr: rate(http_requests_total[1m])

    # Résolution moyenne (5 minutes)
    - record: service:requests:rate5m
      expr: avg_over_time(service:requests:rate1m[5m])

    # Résolution grossière (1 heure)
    - record: service:requests:rate1h
      expr: avg_over_time(service:requests:rate5m[1h])
```

## Monitoring des Recording Rules

### Métriques de santé des règles

Prometheus expose des métriques sur l'exécution des recording rules :

```promql
# Durée d'évaluation des groupes de règles
prometheus_rule_group_duration_seconds

# Dernière évaluation réussie
prometheus_rule_group_last_evaluation_timestamp_seconds

# Nombre d'échecs d'évaluation
prometheus_rule_evaluation_failures_total
```

### Dashboard de monitoring des règles

Créez un dashboard Grafana pour surveiller vos recording rules :

```yaml
# Requêtes pour votre dashboard
# Panel 1: Durée d'évaluation par groupe
rate(prometheus_rule_group_duration_seconds_sum[5m])
/
rate(prometheus_rule_group_duration_seconds_count[5m])

# Panel 2: Règles en échec
increase(prometheus_rule_evaluation_failures_total[1h])

# Panel 3: Fréquence d'évaluation
rate(prometheus_rule_evaluations_total[5m])
```

## Bonnes pratiques et recommandations

### Fréquence d'évaluation

Adaptez l'intervalle d'évaluation selon le type de métrique :
- **Métriques critiques** : 15-30 secondes
- **Métriques de tendance** : 1-5 minutes
- **Agrégations long terme** : 5-15 minutes

### Gestion de la cardinalité

Attention à ne pas créer de rules qui augmentent drastiquement la cardinalité :

```yaml
# ❌ MAUVAIS : Crée trop de séries
- record: pod:cpu:by_container_and_image
  expr: |
    sum by (pod, container, image, namespace, node) (
      rate(container_cpu_usage_seconds_total[5m])
    )

# ✅ BON : Agrégation raisonnable
- record: namespace:cpu:rate5m
  expr: |
    sum by (namespace) (
      rate(container_cpu_usage_seconds_total[5m])
    )
```

### Documentation des règles

Documentez toujours vos recording rules :

```yaml
- record: cluster:api_server:request_rate
  expr: sum(rate(apiserver_request_total[5m]))
  # Description: Taux de requêtes total vers l'API server
  # Utilisé par: Dashboard "Cluster Overview"
  # Créé le: 2024-01-15
  # Auteur: Équipe Platform
```

## Dépannage courant

### Les règles ne s'exécutent pas

Vérifiez que Prometheus charge bien vos règles :
```bash
# Vérifiez les logs de Prometheus
kubectl logs -n observability prometheus-0 | grep "rule"

# Vérifiez la configuration
kubectl describe configmap -n observability prometheus-config
```

### Performance dégradée malgré les rules

Si les performances ne s'améliorent pas :
1. Vérifiez que vos dashboards utilisent bien les nouvelles métriques
2. Assurez-vous que l'intervalle d'évaluation est approprié
3. Contrôlez la complexité des expressions PromQL dans les rules

### Métriques manquantes

Si les métriques créées par les rules n'apparaissent pas :
1. Attendez au moins deux cycles d'évaluation
2. Vérifiez l'absence d'erreurs dans l'expression PromQL
3. Assurez-vous que les métriques sources existent

## Cas d'usage spécifiques pour un lab MicroK8s

### Monitoring des ressources limitées

Dans un lab personnel avec ressources limitées :

```yaml
- name: lab_optimization
  interval: 60s
  rules:
    # Alerte si utilisation CPU > 80%
    - record: lab:high_cpu_usage
      expr: |
        (cluster:cpu_usage_percentage:rate5m > 80)
        * cluster:cpu_usage_percentage:rate5m

    # Identification des pods gourmands
    - record: lab:top_cpu_pods
      expr: |
        topk(5,
          pod:container_cpu_usage:rate5m
        )
```

### Métriques pour apprentissage

Créez des métriques simplifiées pour comprendre Kubernetes :

```yaml
- name: learning_metrics
  rules:
    # Nombre de pods par namespace
    - record: learning:pods_per_namespace
      expr: |
        count by (namespace) (
          up{job="kubelet"}
        )

    # Redémarrages de pods
    - record: learning:pod_restart_rate
      expr: |
        rate(kube_pod_container_status_restarts_total[1h])
```

## Conclusion

Les recording rules sont un outil essentiel pour optimiser votre stack Prometheus dans MicroK8s. Elles permettent non seulement d'améliorer les performances, mais aussi de standardiser vos métriques et de simplifier vos requêtes PromQL. Dans un environnement de lab personnel où les ressources sont précieuses, leur utilisation judicieuse fait la différence entre un système de monitoring réactif et un système lent.

Commencez par identifier vos requêtes les plus fréquentes et complexes, puis créez progressivement des recording rules pour les optimiser. Avec le temps, vous construirez une bibliothèque de métriques pré-calculées qui rendront votre monitoring plus efficace et plus facile à maintenir.

⏭️
