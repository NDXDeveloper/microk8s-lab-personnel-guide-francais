🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 RBAC (Contrôle d'accès basé sur les rôles)

## Qu'est-ce que RBAC ?

RBAC (Role-Based Access Control) est le système de gestion des permissions dans Kubernetes. Imaginez-le comme un système de badges dans un bâtiment : chaque personne reçoit un badge qui détermine dans quelles pièces elle peut entrer et ce qu'elle peut y faire. Dans Kubernetes, RBAC détermine qui peut accéder à quelles ressources et quelles actions ils peuvent effectuer.

### Pourquoi RBAC est essentiel

Sans RBAC, toute personne ayant accès à votre cluster aurait les pleins pouvoirs - comme donner les clés de toutes les portes à chaque visiteur. RBAC vous permet de :

- **Limiter les dégâts** en cas de compromission d'un compte
- **Séparer les responsabilités** entre différentes équipes ou applications
- **Respecter le principe du moindre privilège** : chacun n'a que les permissions nécessaires
- **Auditer précisément** qui fait quoi dans le cluster

## Les concepts fondamentaux de RBAC

RBAC fonctionne avec quatre éléments principaux qui travaillent ensemble :

### 1. Les Sujets (Subjects) - "Qui"

Les sujets sont les entités qui veulent effectuer des actions. Il existe trois types :

**Users (Utilisateurs)**
- Représentent des personnes réelles
- Gérés en dehors de Kubernetes (certificats, tokens, etc.)
- Exemple : "jean@monlab.local"

**ServiceAccounts**
- Représentent des applications ou processus
- Gérés directement par Kubernetes
- Vivent dans un namespace spécifique
- Exemple : une application qui a besoin de lire des ConfigMaps

**Groups (Groupes)**
- Collections d'utilisateurs
- Permettent de gérer plusieurs utilisateurs ensemble
- Exemple : "system:authenticated" pour tous les utilisateurs authentifiés

### 2. Les Ressources - "Quoi"

Les ressources sont les objets Kubernetes sur lesquels on veut agir :

- **Pods** : les conteneurs en cours d'exécution
- **Services** : les points d'accès réseau
- **ConfigMaps** : les configurations
- **Secrets** : les données sensibles
- **Deployments** : les définitions d'applications
- Et bien d'autres...

### 3. Les Verbes - "Actions"

Les verbes définissent ce qu'on peut faire avec les ressources :

- **get** : lire une ressource spécifique
- **list** : lister toutes les ressources d'un type
- **watch** : surveiller les changements
- **create** : créer de nouvelles ressources
- **update** : modifier des ressources existantes
- **patch** : modifier partiellement
- **delete** : supprimer des ressources

### 4. Les Rôles et Bindings - "Comment"

C'est ici que tout se connecte :

**Role / ClusterRole**
- Définit un ensemble de permissions
- Role : limité à un namespace
- ClusterRole : valable sur tout le cluster

**RoleBinding / ClusterRoleBinding**
- Attache un rôle à des sujets
- RoleBinding : dans un namespace
- ClusterRoleBinding : sur tout le cluster

## RBAC dans MicroK8s

### Activation de RBAC

Dans MicroK8s, RBAC est généralement activé par défaut dans les versions récentes. Pour vérifier :

```bash
# Vérifier si RBAC est activé
microk8s status | grep rbac

# Si ce n'est pas le cas, l'activer
microk8s enable rbac

# Vérifier que l'autorisation est bien en mode RBAC
microk8s kubectl describe pod kube-apiserver -n kube-system | grep authorization-mode
```

### Structure par défaut

MicroK8s crée automatiquement plusieurs rôles et bindings essentiels :

```bash
# Voir les ClusterRoles par défaut
microk8s kubectl get clusterroles

# Voir les ClusterRoleBindings par défaut
microk8s kubectl get clusterrolebindings

# Explorer un rôle système important
microk8s kubectl describe clusterrole cluster-admin
```

## Anatomie d'un Role

Un Role définit des permissions dans un namespace. Voici sa structure :

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: mon-app  # Le namespace où ce rôle s'applique
  name: pod-reader    # Nom descriptif du rôle
rules:
- apiGroups: [""]     # "" = core API group
  resources: ["pods"] # Les ressources concernées
  verbs: ["get", "list", "watch"] # Les actions autorisées
```

### Décortiquons chaque partie

**apiGroups**
- `[""]` : Le groupe API core (pods, services, etc.)
- `["apps"]` : Pour deployments, replicasets
- `["batch"]` : Pour les jobs et cronjobs
- `["*"]` : Tous les groupes (à utiliser avec précaution)

**resources**
- Liste les types de ressources Kubernetes
- Peut inclure des sous-ressources : `["pods/log"]`
- `["*"]` pour toutes les ressources (dangereux)

**verbs**
- Les actions autorisées sur ces ressources
- Peuvent être cumulés : `["get", "list", "watch"]`
- `["*"]` pour toutes les actions (très dangereux)

## ClusterRole vs Role

### Role : Permissions locales

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development  # Limité à ce namespace
  name: dev-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "update", "delete"]
```

Ce rôle ne fonctionne QUE dans le namespace "development".

### ClusterRole : Permissions globales

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole  # Pas de namespace ici
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

Ce ClusterRole peut être utilisé :
- Dans tout le cluster avec un ClusterRoleBinding
- Dans un namespace spécifique avec un RoleBinding

## Les Bindings : Attacher les permissions

### RoleBinding : Attribution locale

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: development
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: mon-app-sa
  namespace: development
roleRef:
  kind: Role
  name: pod-reader  # Le rôle créé précédemment
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding : Attribution globale

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-secret-reader
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## ServiceAccounts : Identités pour les applications

Les ServiceAccounts sont cruciaux pour les applications qui tournent dans Kubernetes :

### Création d'un ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mon-app-sa
  namespace: production
```

### Utilisation dans un Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-app-pod
  namespace: production
spec:
  serviceAccountName: mon-app-sa  # Utilise ce ServiceAccount
  containers:
  - name: app
    image: mon-image:latest
```

### ServiceAccount par défaut

Chaque namespace a un ServiceAccount "default" créé automatiquement :

```bash
# Voir les ServiceAccounts d'un namespace
microk8s kubectl get serviceaccounts -n default

# Détails du ServiceAccount default
microk8s kubectl describe sa default -n default
```

## Scénarios pratiques courants

### Scénario 1 : Développeur avec accès limité

Un développeur doit pouvoir déployer dans le namespace "dev" mais seulement consulter en "prod" :

```yaml
# Role pour le namespace dev
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: developer-role
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]
---
# ClusterRole pour lecture seule
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
# Binding pour dev
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
subjects:
- kind: User
  name: developer@monlab.local
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
---
# Binding pour prod (lecture seule)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-readonly-binding
  namespace: prod
subjects:
- kind: User
  name: developer@monlab.local
roleRef:
  kind: ClusterRole  # On utilise un ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```

### Scénario 2 : Application avec accès aux ConfigMaps

Une application qui doit lire ses configurations :

```yaml
# ServiceAccount pour l'application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-reader-sa
  namespace: apps
---
# Role pour lire les ConfigMaps
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: apps
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-configs
  namespace: apps
subjects:
- kind: ServiceAccount
  name: config-reader-sa
  namespace: apps
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

### Scénario 3 : Monitoring avec accès cluster-wide

Un système de monitoring qui doit voir toutes les métriques :

```yaml
# ServiceAccount pour Prometheus
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
# ClusterRole pour accès aux métriques
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-scraper
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - nodes/metrics
  verbs: ["get"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-scraper
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-scraper
  apiGroup: rbac.authorization.k8s.io
```

## Commandes utiles pour gérer RBAC

### Vérifier les permissions

```bash
# Vérifier ses propres permissions
microk8s kubectl auth can-i create pods

# Vérifier dans un namespace spécifique
microk8s kubectl auth can-i delete deployments -n production

# Vérifier toutes ses permissions
microk8s kubectl auth can-i --list

# Vérifier les permissions d'un ServiceAccount
microk8s kubectl auth can-i delete pods --as=system:serviceaccount:default:mon-sa

# Vérifier les permissions d'un utilisateur
microk8s kubectl auth can-i get secrets --as=alice@monlab.local
```

### Explorer les rôles existants

```bash
# Lister tous les rôles d'un namespace
microk8s kubectl get roles -n development

# Voir les détails d'un rôle
microk8s kubectl describe role pod-reader -n development

# Exporter un rôle en YAML
microk8s kubectl get role pod-reader -n development -o yaml

# Lister tous les ClusterRoles
microk8s kubectl get clusterroles

# Voir qui a quoi (bindings)
microk8s kubectl get rolebindings -n development
microk8s kubectl get clusterrolebindings
```

### Débugger les problèmes RBAC

```bash
# Voir les événements liés aux erreurs d'autorisation
microk8s kubectl get events -n kube-system | grep -i forbidden

# Tester avec impersonation
microk8s kubectl get pods --as=system:serviceaccount:default:mon-sa

# Voir les détails d'un ServiceAccount et ses tokens
microk8s kubectl describe sa mon-sa -n default

# Identifier quel binding donne une permission
microk8s kubectl get rolebindings,clusterrolebindings -A -o wide | grep alice
```

## Bonnes pratiques RBAC

### 1. Principe du moindre privilège

**À faire :**
- Commencer avec zéro permission et ajouter au besoin
- Créer des rôles spécifiques plutôt que réutiliser admin
- Limiter au maximum l'usage de wildcards (*)

**À éviter :**
```yaml
# DANGEREUX - Trop de permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### 2. Organisation des rôles

**Structure recommandée :**
- Un rôle par fonction (lecteur, éditeur, admin)
- Des rôles par type de ressource si nécessaire
- Utiliser les ClusterRoles pour les permissions réutilisables

```yaml
# Bon exemple : rôle spécifique et clair
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # Lecture seule sur les pods
```

### 3. Namespaces comme frontières de sécurité

**Stratégie de namespaces :**
```
production/     # Accès très restreint
├── admin       # ClusterRoleBinding admin
└── deployer    # RoleBinding limité

staging/        # Accès modéré
├── developer   # RoleBinding avec plus de droits
└── tester      # RoleBinding lecture + logs

development/    # Accès plus ouvert
└── developer   # RoleBinding quasi-admin
```

### 4. ServiceAccounts dédiés

**Un ServiceAccount par application :**
```yaml
# Mauvais : utiliser default
spec:
  # serviceAccountName non spécifié = default

# Bon : ServiceAccount dédié
spec:
  serviceAccountName: mon-app-specific-sa
```

### 5. Audit et revue régulière

**Checklist de maintenance :**
- Revoir les ClusterRoleBindings mensuellement
- Vérifier les comptes inutilisés
- Auditer les permissions avec wildcards
- Documenter pourquoi chaque binding existe

## Rôles prédéfinis utiles

Kubernetes fournit des ClusterRoles prédéfinis :

### view (Lecture seule)
```bash
microk8s kubectl describe clusterrole view
```
Permet de voir la plupart des ressources mais pas les secrets.

### edit (Éditeur)
```bash
microk8s kubectl describe clusterrole edit
```
Permet de modifier la plupart des ressources sauf RBAC et quotas.

### admin (Administrateur)
```bash
microk8s kubectl describe clusterrole admin
```
Contrôle total sur un namespace sauf quotas et le namespace lui-même.

### cluster-admin (Super Admin)
```bash
microk8s kubectl describe clusterrole cluster-admin
```
Contrôle total sur tout le cluster - À utiliser avec extrême précaution !

## Exemple complet : Mise en place RBAC pour une équipe

Voici un exemple complet pour configurer RBAC pour une petite équipe :

```yaml
# 1. Créer les namespaces
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha-dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha-prod
---

# 2. ServiceAccounts pour les applications
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-backend
  namespace: team-alpha-dev
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-backend
  namespace: team-alpha-prod
---

# 3. Role pour développeurs en dev
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-alpha-dev
  name: developer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---

# 4. Role pour développeurs en prod (lecture seule)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-alpha-prod
  name: viewer
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
---

# 5. Role pour l'application backend
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-alpha-dev
  name: backend-permissions
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
---

# 6. RoleBindings
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-dev
  namespace: team-alpha-dev
subjects:
- kind: Group
  name: team-alpha-devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers-prod
  namespace: team-alpha-prod
subjects:
- kind: Group
  name: team-alpha-devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: viewer
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-binding
  namespace: team-alpha-dev
subjects:
- kind: ServiceAccount
  name: app-backend
  namespace: team-alpha-dev
roleRef:
  kind: Role
  name: backend-permissions
  apiGroup: rbac.authorization.k8s.io
```

## Dépannage des problèmes RBAC courants

### Problème 1 : "Forbidden" errors

**Symptôme :**
```
Error from server (Forbidden): pods is forbidden: User "alice" cannot list resource "pods" in API group "" in the namespace "production"
```

**Diagnostic :**
```bash
# Vérifier les permissions de l'utilisateur
microk8s kubectl auth can-i list pods -n production --as=alice

# Lister les bindings de l'utilisateur
microk8s kubectl get rolebindings,clusterrolebindings -A | grep alice
```

### Problème 2 : ServiceAccount ne fonctionne pas

**Vérifications :**
```bash
# Le ServiceAccount existe-t-il ?
microk8s kubectl get sa mon-sa -n default

# Le pod utilise-t-il le bon ServiceAccount ?
microk8s kubectl get pod mon-pod -o jsonpath='{.spec.serviceAccountName}'

# Les bindings sont-ils corrects ?
microk8s kubectl get rolebindings -n default -o yaml | grep -A5 mon-sa
```

### Problème 3 : Permissions héritées inattendues

**Investigation :**
```bash
# Vérifier les groupes de l'utilisateur
microk8s kubectl auth can-i --list --as=alice

# Vérifier les ClusterRoleBindings globaux
microk8s kubectl get clusterrolebindings -o wide

# Chercher des bindings avec des groupes système
microk8s kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.name | contains("system:"))'
```

## Résumé et points clés

RBAC est le gardien de votre cluster Kubernetes. Dans votre lab MicroK8s :

1. **Activez RBAC** dès le début, même pour un lab personnel
2. **Créez des ServiceAccounts dédiés** pour chaque application
3. **Utilisez le principe du moindre privilège** - commencez restrictif
4. **Organisez avec des namespaces** - ce sont vos frontières de sécurité
5. **Réutilisez les ClusterRoles prédéfinis** quand possible
6. **Testez les permissions** avec `kubectl auth can-i`
7. **Documentez vos décisions** RBAC pour faciliter la maintenance

RBAC peut sembler complexe au début, mais c'est un investissement essentiel pour la sécurité de votre cluster. Commencez simple, avec quelques rôles basiques, et affinez progressivement selon vos besoins.

---

*Prochain sujet : 11.2 Network Policies - Comment isoler et contrôler le trafic réseau entre vos pods*

⏭️
