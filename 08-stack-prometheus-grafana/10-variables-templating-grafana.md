🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.10 Variables et Templating dans Grafana

## Introduction

Les variables et le templating dans Grafana sont des fonctionnalités puissantes qui permettent de créer des dashboards dynamiques et réutilisables. Au lieu d'avoir des valeurs codées en dur dans vos requêtes, vous pouvez utiliser des variables qui peuvent être modifiées facilement via des menus déroulants en haut de votre dashboard.

Cette approche est particulièrement utile dans un environnement Kubernetes où vous avez plusieurs namespaces, pods, nodes, et services à surveiller. Imaginez devoir créer un dashboard séparé pour chaque namespace - avec les variables, un seul dashboard suffit !

## Pourquoi utiliser des variables ?

### Avantages principaux

**Réutilisabilité** : Un seul dashboard peut servir pour plusieurs environnements, applications ou composants. Par exemple, le même dashboard peut afficher les métriques de n'importe quel pod simplement en changeant une sélection dans un menu déroulant.

**Maintenance simplifiée** : Lorsque vous devez modifier une requête, vous ne le faites qu'une fois au lieu de la dupliquer dans plusieurs panneaux. Si votre structure de métriques change, vous n'avez qu'un endroit à mettre à jour.

**Expérience utilisateur améliorée** : Les utilisateurs peuvent explorer les données sans avoir besoin de connaître PromQL ou de modifier les requêtes. Ils utilisent simplement les menus déroulants pour filtrer ce qu'ils veulent voir.

**Dashboards interactifs** : Les variables peuvent dépendre les unes des autres, créant des filtres en cascade. Par exemple, sélectionner un namespace peut automatiquement mettre à jour la liste des pods disponibles.

## Types de variables dans Grafana

### Variables de type Query

Ce sont les plus courantes. Elles exécutent une requête Prometheus pour récupérer dynamiquement une liste de valeurs.

```promql
# Exemple : récupérer tous les namespaces
label_values(kube_pod_info, namespace)
```

Cette requête retournera tous les namespaces qui ont des pods actifs. La variable sera automatiquement mise à jour si de nouveaux namespaces sont créés.

### Variables de type Custom

Parfaites quand vous avez une liste fixe de valeurs que vous connaissez à l'avance.

```
Valeurs : production,staging,development
```

Utile pour des environnements fixes ou des options de configuration qui ne changent pas souvent.

### Variables de type Constant

Une valeur unique qui ne change pas mais que vous voulez centraliser pour la maintenance.

```
Valeur : mon-cluster-k8s
```

Si vous devez référencer cette valeur dans 50 requêtes, la changer à un seul endroit met tout à jour.

### Variables de type Datasource

Permet de changer dynamiquement la source de données Prometheus.

```
Type : Prometheus
Regex : /.*prometheus.*/
```

Très utile si vous avez plusieurs instances Prometheus (dev, staging, prod) et que vous voulez basculer entre elles facilement.

### Variables de type Interval

Définit des intervalles de temps pour l'agrégation des données.

```
Valeurs : 1m,5m,10m,30m,1h
```

Permet aux utilisateurs d'ajuster la granularité des métriques affichées selon leurs besoins.

### Variables de type Text Box

Permet à l'utilisateur d'entrer du texte libre.

```
Valeur par défaut : app-
```

Utile pour des recherches ou des filtres personnalisés que l'utilisateur peut définir à la volée.

## Création d'une variable pas à pas

### Étape 1 : Accéder aux paramètres

Dans votre dashboard Grafana, cliquez sur l'icône d'engrenage (⚙️) en haut à droite pour accéder aux paramètres du dashboard. Dans le menu de gauche, sélectionnez "Variables".

### Étape 2 : Ajouter une nouvelle variable

Cliquez sur "Add variable" ou "New" selon votre version de Grafana.

### Étape 3 : Configuration de base

**Name** : Le nom de votre variable (sans espaces, utilisé dans les requêtes)
```
namespace
```

**Label** : Le nom affiché dans le dashboard (peut contenir des espaces)
```
Namespace K8s
```

**Type** : Sélectionnez "Query" pour une variable dynamique

### Étape 4 : Configuration de la requête

**Data source** : Sélectionnez votre instance Prometheus

**Query** : Entrez votre requête PromQL
```promql
label_values(kube_pod_info, namespace)
```

**Regex** : (Optionnel) Pour filtrer les résultats
```
/^prod-.*/
```
Ceci ne gardera que les namespaces commençant par "prod-"

### Étape 5 : Options d'affichage

**Sort** : Comment trier les valeurs
- Disabled : Pas de tri
- Alphabetical (asc) : Ordre alphabétique croissant
- Alphabetical (desc) : Ordre alphabétique décroissant
- Numerical (asc) : Ordre numérique croissant
- Numerical (desc) : Ordre numérique décroissant

**Multi-value** : Permet de sélectionner plusieurs valeurs
✅ Coché si vous voulez permettre la sélection multiple

**Include All option** : Ajoute une option "All"
✅ Recommandé pour permettre de voir toutes les valeurs d'un coup

### Étape 6 : Valeur par défaut

**Custom all value** : Valeur utilisée quand "All" est sélectionné
```
.*
```
Une regex qui matche tout

## Utilisation des variables dans les requêtes

### Syntaxe de base

Dans vos requêtes PromQL, utilisez la syntaxe `$variable_name` ou `${variable_name}` :

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="$namespace"}[5m])) by (pod)
```

### Utilisation avec multi-value

Quand vous permettez la sélection multiple, utilisez la syntaxe avec regex :

```promql
sum(rate(container_cpu_usage_seconds_total{namespace=~"$namespace"}[5m])) by (pod)
```

Notez le `=~` au lieu de `=` pour supporter les regex et donc les sélections multiples.

### Variables dans les titres de panneaux

Vous pouvez aussi utiliser les variables dans les titres :

```
CPU Usage - Namespace: $namespace
```

Le titre se mettra à jour automatiquement selon la sélection.

## Variables en cascade (dépendantes)

### Concept

Les variables peuvent dépendre les unes des autres. Par exemple, la liste des pods dépend du namespace sélectionné.

### Variable 1 : Namespace

```promql
label_values(kube_pod_info, namespace)
```

### Variable 2 : Pod (dépendant du namespace)

```promql
label_values(kube_pod_info{namespace="$namespace"}, pod)
```

Maintenant, quand vous changez le namespace, la liste des pods se met à jour automatiquement !

### Variable 3 : Container (dépendant du pod)

```promql
label_values(kube_pod_container_info{namespace="$namespace",pod="$pod"}, container)
```

Cette cascade permet une navigation intuitive dans vos ressources Kubernetes.

## Variables avancées avec Regex

### Extraction de parties de labels

Supposons que vos pods ont des noms comme `app-frontend-7b9d8c-x2fg4` et vous voulez extraire juste la partie `frontend` :

```promql
label_values(kube_pod_info, pod)
```

Avec le Regex :
```
/app-([^-]+)-.*/
```

### Filtrage intelligent

Pour ne garder que les pods de production (supposant qu'ils ont "prod" dans leur nom) :

```
/.*prod.*/
```

### Transformation de valeurs

Vous pouvez transformer les valeurs récupérées. Par exemple, remplacer les underscores par des tirets :

Query : `label_values(metric_name, label)`
Regex : `/_/-/g`

## Variables de répétition (Repeat)

### Concept de répétition

Grafana peut automatiquement dupliquer des panneaux ou des lignes pour chaque valeur d'une variable.

### Configuration d'un panneau répétitif

1. Éditez votre panneau
2. Dans l'onglet "General", trouvez "Repeat options"
3. Sélectionnez la variable à utiliser pour la répétition
4. Choisissez la direction : Horizontal ou Vertical

### Exemple pratique

Si vous avez une variable `$node` avec 3 nodes, et vous configurez un panneau pour se répéter sur cette variable, Grafana créera automatiquement 3 panneaux, un pour chaque node.

### Répétition de lignes

Même principe mais au niveau d'une ligne entière de panneaux :
1. Survolez le titre de la ligne
2. Cliquez sur l'icône d'engrenage
3. Dans "Repeat for", sélectionnez votre variable

## Variables d'annotations

### Utilité

Les variables peuvent contrôler l'affichage des annotations (marqueurs d'événements sur vos graphiques).

### Configuration

```promql
changes(kube_deployment_status_replicas{namespace="$namespace"}[5m]) > 0
```

Cette requête affichera une annotation chaque fois que le nombre de replicas change dans le namespace sélectionné.

## Bonnes pratiques

### Nommage cohérent

Utilisez une convention de nommage claire :
- `k8s_namespace` pour les namespaces Kubernetes
- `k8s_pod` pour les pods
- `prom_job` pour les jobs Prometheus

### Documentation des variables

Dans le champ "Description" de chaque variable, expliquez :
- Ce que représente la variable
- D'où viennent les données
- Les dépendances éventuelles

Exemple :
```
Sélectionne le namespace Kubernetes à surveiller.
Source : label 'namespace' de kube_pod_info
Actualisation : Toutes les 5 minutes
```

### Performance

**Limitez le nombre de valeurs** : Une variable avec 1000+ valeurs sera lente
```promql
label_values(kube_pod_info{namespace="production"}, pod)
```
Mieux que :
```promql
label_values(kube_pod_info, pod)
```

**Utilisez le cache** : Dans les options de la variable, activez "Refresh: On Dashboard Load" plutôt que "On Time Range Change" si les valeurs ne changent pas souvent.

### Valeurs par défaut intelligentes

Configurez toujours une valeur par défaut pertinente :
- Pour un namespace : "default" ou "production"
- Pour un intervalle : "5m"
- Pour un node : utilisez "All" si disponible

### Gestion des erreurs

Prévoyez le cas où aucune donnée n'est disponible :

```promql
label_values(kube_pod_info, namespace) or vector(0)
```

## Cas d'usage concrets dans MicroK8s

### Dashboard multi-namespace

Variable namespace :
```promql
label_values(kube_namespace_created, namespace)
```

Utilisation dans les requêtes :
```promql
sum(kube_pod_container_status_running{namespace="$namespace"}) by (pod)
```

### Surveillance par application

Variable pour les labels d'application :
```promql
label_values(kube_pod_labels, label_app)
```

Requête pour CPU par app :
```promql
sum(rate(container_cpu_usage_seconds_total{pod=~".*",label_app="$app"}[5m])) by (pod)
```

### Analyse temporelle flexible

Variable d'intervalle :
```
1m,5m,15m,30m,1h,3h,6h,12h,1d
```

Utilisation :
```promql
increase(http_requests_total{namespace="$namespace"}[$interval])
```

## Dépannage courant

### La variable ne montre aucune valeur

**Vérifiez la requête** : Testez-la directement dans Prometheus
```promql
label_values(kube_pod_info, namespace)
```

**Vérifiez la source de données** : Assurez-vous que la bonne instance Prometheus est sélectionnée

**Vérifiez les permissions** : L'utilisateur Grafana a-t-il accès aux métriques ?

### Les valeurs ne se mettent pas à jour

**Vérifiez le refresh** : Dans les options de la variable, configurez le rafraîchissement approprié

**Videz le cache** : Parfois, un simple refresh du dashboard (F5) résout le problème

### La sélection "All" ne fonctionne pas

**Vérifiez la syntaxe** : Utilisez `=~` au lieu de `=` dans vos requêtes
```promql
{namespace=~"$namespace"}  # Correct
{namespace="$namespace"}    # Incorrect pour multi-value
```

**Vérifiez la Custom all value** : Devrait être `.*` ou une regex appropriée

### Performance dégradée

**Trop de valeurs** : Limitez avec des filtres
```promql
label_values(kube_pod_info{namespace=~"prod.*"}, pod)
```

**Requêtes trop complexes** : Simplifiez ou utilisez des recording rules Prometheus

## Conclusion

Les variables et le templating transforment des dashboards statiques en outils d'exploration dynamiques. Dans un environnement MicroK8s, où vous gérez potentiellement plusieurs applications et namespaces, cette fonctionnalité devient indispensable pour maintenir des dashboards maintenables et utilisables.

Commencez simple avec une ou deux variables, puis construisez progressivement des dashboards plus sophistiqués. N'oubliez pas que l'objectif est de rendre l'information accessible et explorable, pas de créer de la complexité inutile.

Les variables bien configurées permettent à n'importe qui dans votre équipe d'explorer les métriques sans connaître PromQL, rendant l'observabilité accessible à tous.

⏭️
