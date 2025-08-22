🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 08. Stack Prometheus et Grafana

## Vue d'ensemble

La mise en place d'une stack de monitoring est cruciale pour tout environnement Kubernetes, même dans un contexte de lab personnel. Prometheus et Grafana forment le duo de référence pour l'observabilité dans l'écosystème Kubernetes, offrant une solution complète de collecte de métriques, de stockage, de requêtage et de visualisation.

## Pourquoi Prometheus et Grafana ?

### Prometheus : Le cœur du monitoring

Prometheus s'est imposé comme la solution standard de monitoring pour Kubernetes grâce à plusieurs atouts majeurs :

- **Modèle de données multi-dimensionnel** : Les métriques sont identifiées par leur nom et des paires clé/valeur (labels), permettant une granularité fine dans l'analyse
- **Pull model natif** : Prometheus récupère activement les métriques depuis les targets, simplifiant la configuration réseau et la découverte de services
- **Service discovery Kubernetes** : Intégration native avec les APIs Kubernetes pour découvrir automatiquement pods, services et nodes
- **Langage de requête puissant** : PromQL permet des analyses complexes et des agrégations sophistiquées
- **Stockage optimisé** : Base de données de séries temporelles (TSDB) conçue spécifiquement pour les métriques
- **Écosystème riche** : Large gamme d'exporters disponibles pour monitorer pratiquement n'importe quel système

### Grafana : La fenêtre sur vos métriques

Grafana complète parfaitement Prometheus en offrant :

- **Visualisations riches** : Graphiques, jauges, heatmaps, et dizaines d'autres types de visualisation
- **Dashboards dynamiques** : Création de tableaux de bord interactifs avec variables et filtres
- **Multi-datasources** : Capacité de combiner données de Prometheus avec d'autres sources
- **Alerting visuel** : Configuration intuitive des seuils et règles d'alerte
- **Écosystème de dashboards** : Milliers de dashboards communautaires prêts à l'emploi
- **Gestion des équipes** : Permissions et organisation pour environnements multi-utilisateurs

## Architecture dans MicroK8s

Dans un environnement MicroK8s, la stack monitoring s'intègre naturellement avec l'architecture existante :

```
┌───────────────────────────────────────────────────────┐
│                    MicroK8s Cluster                   │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ┌──────────────┐     ┌──────────────┐                │
│  │  Prometheus  │────▶│   Targets    │                │
│  │   Server     │     │              │                │
│  └──────────────┘     │ - Nodes      │                │
│         │             │ - Pods       │                │
│         │             │ - Services   │                │
│         ▼             │ - Ingress    │                │
│  ┌──────────────┐     └──────────────┘                │
│  │   Storage    │                                     │
│  │    (TSDB)    │     ┌──────────────┐                │
│  └──────────────┘     │  Exporters   │                │
│         │             │              │                │
│         │             │ - Node       │                │
│         ▼             │ - kube-state │                │
│  ┌──────────────┐     │ - cAdvisor   │                │
│  │   Grafana    │     │ - Custom     │                │
│  │              │     └──────────────┘                │
│  └──────────────┘                                     │
│         │                                             │
│         ▼                                             │
│  ┌──────────────┐                                     │
│  │   Ingress    │                                     │
│  │  Controller  │                                     │
│  └──────────────┘                                     │
│                                                       │
└───────────────────────────────────────────────────────┘
           │
           ▼
    [Accès externe]
    https://grafana.votredomaine.com
```

## Cas d'usage dans un lab personnel

### Monitoring d'infrastructure
- **Santé du cluster** : CPU, mémoire, réseau et stockage des nodes
- **Ressources Kubernetes** : Utilisation par namespace, pod, container
- **Performance réseau** : Latence, throughput, erreurs
- **Stockage** : Volumes persistants, utilisation disque

### Monitoring applicatif
- **Métriques métier** : Requêtes par seconde, temps de réponse, taux d'erreur
- **Métriques techniques** : Pools de connexions, garbage collection, threads
- **SLI/SLO** : Indicateurs de niveau de service pour vos applications
- **Debugging** : Analyse post-mortem des incidents

### Apprentissage et expérimentation
- **Compréhension de Kubernetes** : Visualiser l'impact des actions sur le cluster
- **Tests de charge** : Observer le comportement sous stress
- **Optimisation** : Identifier les goulots d'étranglement
- **Certification** : Pratique pour CKA/CKAD avec monitoring réel

## Prérequis et considérations

### Ressources système
Pour un lab personnel avec Prometheus et Grafana, prévoir :
- **CPU additionnel** : +0.5 à 1 core pour Prometheus et Grafana combinés
- **RAM supplémentaire** : +1-2 GB minimum (selon la rétention et le nombre de métriques)
- **Stockage** : 10-50 GB pour les données de métriques (selon la rétention souhaitée)

### Configuration réseau
- **Ingress configuré** : Pour accéder à Grafana depuis l'extérieur
- **DNS fonctionnel** : Pour la résolution des services internes
- **Certificats SSL** : Pour sécuriser l'accès à Grafana

### Compétences requises
- **Kubernetes basics** : Comprendre pods, services, deployments
- **YAML** : Pour éditer les configurations
- **Métriques** : Notions de base (gauges, counters, histograms)
- **SQL-like** : PromQL s'apprend facilement avec des bases SQL

## Stratégie de déploiement

### Options disponibles sur MicroK8s

MicroK8s offre plusieurs approches pour déployer la stack monitoring :

1. **Addon observability intégré** : Solution clé en main avec configuration par défaut
2. **Helm charts officiels** : Plus de contrôle sur la configuration
3. **Kube-prometheus-stack** : Stack complète avec opérateurs
4. **Installation manuelle** : Contrôle total mais plus complexe

### Approche recommandée pour un lab

Pour un lab personnel, nous recommandons de commencer avec l'addon observability de MicroK8s pour une mise en route rapide, puis d'évoluer vers une installation personnalisée via Helm pour plus de flexibilité. Cette approche permet :

- **Démarrage rapide** : Monitoring opérationnel en quelques minutes
- **Apprentissage progressif** : Comprendre les composants un par un
- **Personnalisation** : Adapter selon les besoins spécifiques
- **Réversibilité** : Possibilité de revenir en arrière facilement

## Objectifs de cette section

À la fin de cette section, vous serez capable de :

- ✅ Installer et configurer Prometheus dans MicroK8s
- ✅ Déployer Grafana et le connecter à Prometheus
- ✅ Comprendre l'architecture de collecte de métriques
- ✅ Créer des requêtes PromQL pour analyser vos données
- ✅ Construire des dashboards personnalisés dans Grafana
- ✅ Configurer des alertes basées sur les métriques
- ✅ Implémenter des exporters pour vos applications
- ✅ Optimiser les performances et la rétention
- ✅ Mettre en place une stratégie de monitoring complète

## Plan de progression

La section est organisée pour une montée en compétence progressive :

**Phases d'apprentissage :**

1. **Foundation** (8.01-8.05) : Comprendre les concepts et installer les composants de base
2. **Visualization** (8.06-8.10) : Maîtriser Grafana et créer des dashboards efficaces
3. **Alerting** (8.11-8.12) : Mettre en place un système d'alerte proactif
4. **Optimization** (8.13-8.14) : Optimiser pour la production et la haute disponibilité

Chaque sous-section inclura des exemples concrets, des configurations YAML commentées, et des conseils basés sur les meilleures pratiques de l'industrie.

---

*Prêt à transformer votre lab MicroK8s en un environnement fully observable ? Commençons par découvrir l'écosystème Prometheus dans la section suivante.*

⏭️
