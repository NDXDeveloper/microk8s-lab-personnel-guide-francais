🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Section 11 : Sécurité dans MicroK8s

## Introduction - Pourquoi la sécurité est cruciale même dans un lab personnel

La sécurité dans Kubernetes n'est pas uniquement une préoccupation pour les environnements de production. Même dans un laboratoire personnel MicroK8s, l'adoption de bonnes pratiques de sécurité dès le début vous permettra de :

- **Développer des réflexes sécuritaires** qui vous serviront dans des environnements professionnels
- **Protéger votre réseau domestique** contre d'éventuelles compromissions
- **Éviter que votre lab devienne un point d'entrée** pour des attaques sur d'autres systèmes
- **Apprendre et maîtriser** les concepts de sécurité Kubernetes dans un environnement contrôlé
- **Préparer vos applications** à être déployées de manière sécurisée en production

## Vue d'ensemble de la sécurité Kubernetes

Kubernetes offre plusieurs couches de sécurité qui travaillent ensemble pour créer une défense en profondeur. Dans cette section, nous allons explorer les mécanismes essentiels pour sécuriser votre cluster MicroK8s.

### Les 4 piliers de la sécurité Kubernetes

**1. Authentification et Autorisation**
- Qui peut accéder au cluster ?
- Que peuvent-ils faire une fois connectés ?
- Comment gérer les permissions de manière granulaire ?

**2. Isolation et Segmentation**
- Comment isoler les workloads entre eux ?
- Comment limiter les communications réseau ?
- Comment restreindre les capacités des conteneurs ?

**3. Protection des Données Sensibles**
- Comment stocker et gérer les secrets de manière sécurisée ?
- Comment chiffrer les données au repos et en transit ?
- Comment auditer l'accès aux informations sensibles ?

**4. Conformité et Audit**
- Comment suivre qui fait quoi dans le cluster ?
- Comment détecter les comportements anormaux ?
- Comment maintenir la conformité avec les standards de sécurité ?

## Architecture de sécurité dans MicroK8s

MicroK8s simplifie certains aspects de la sécurité tout en maintenant les fonctionnalités essentielles. Voici comment la sécurité est structurée dans votre lab :

### Composants de sécurité natifs

**API Server**
- Point d'entrée central pour toutes les opérations
- Gère l'authentification et l'autorisation
- Applique les politiques d'admission

**etcd**
- Stocke tous les secrets et configurations du cluster
- Doit être protégé et sauvegardé régulièrement
- Chiffrement au repos disponible

**kubelet**
- Applique les politiques de sécurité au niveau des pods
- Gère les volumes et les secrets montés
- Vérifie les signatures d'images (optionnel)

**Controller Manager**
- Gère les ServiceAccounts et leurs tokens
- Applique les quotas de ressources
- Maintient l'état désiré du cluster

### Spécificités MicroK8s

MicroK8s apporte quelques particularités en matière de sécurité :

- **Confinement Snap** : MicroK8s s'exécute dans un environnement Snap isolé, ajoutant une couche de sécurité supplémentaire
- **Certificats auto-générés** : Les certificats TLS sont automatiquement créés et gérés
- **Addons sécurisés** : Les addons comme RBAC peuvent être activés facilement avec `microk8s enable`
- **Mise à jour simplifiée** : Les mises à jour de sécurité sont gérées via le système Snap

## Menaces courantes et leurs mitigations

### Menaces externes

**1. Exposition non intentionnelle de services**
- **Risque** : Services accessibles depuis Internet sans protection
- **Mitigation** : Utiliser des Ingress avec authentification, limiter les NodePorts

**2. Attaques sur l'API Server**
- **Risque** : Accès non autorisé aux ressources du cluster
- **Mitigation** : RBAC strict, authentification forte, limitation des accès réseau

**3. Images de conteneurs compromises**
- **Risque** : Exécution de code malveillant dans le cluster
- **Mitigation** : Scanner les images, utiliser des registres privés, signatures d'images

### Menaces internes

**1. Escalade de privilèges**
- **Risque** : Un pod compromis obtient plus de permissions
- **Mitigation** : Pod Security Standards, principes du moindre privilège

**2. Mouvements latéraux**
- **Risque** : Propagation d'une compromission entre pods
- **Mitigation** : Network Policies, segmentation des namespaces

**3. Fuite de secrets**
- **Risque** : Exposition de mots de passe, clés API, certificats
- **Mitigation** : Encryption at rest, rotation des secrets, outils de gestion des secrets

## Stratégie de sécurité progressive

Pour un lab personnel, nous recommandons une approche progressive de la sécurité :

### Phase 1 : Fondations (Immédiat)
- Activer RBAC
- Configurer les namespaces de base
- Sécuriser l'accès à kubectl
- Mettre en place les premiers ServiceAccounts

### Phase 2 : Renforcement (Semaine 1-2)
- Implémenter les Network Policies de base
- Configurer Pod Security Standards
- Mettre en place la gestion des secrets
- Activer l'audit logging basique

### Phase 3 : Optimisation (Mois 1)
- Scanner les images de conteneurs
- Implémenter des politiques d'admission
- Affiner les Network Policies
- Automatiser la rotation des secrets

### Phase 4 : Maturité (Continu)
- Monitoring de sécurité avancé
- Tests de pénétration réguliers
- Conformité avec les frameworks (CIS Benchmark)
- Incident response planning

## Outils et ressources essentiels

### Outils intégrés à MicroK8s

```bash
# Activer les fonctionnalités de sécurité essentielles
microk8s enable rbac
microk8s enable dns
microk8s enable ingress

# Vérifier le statut de sécurité
microk8s status
microk8s inspect
```

### Outils externes recommandés

**Pour l'analyse de sécurité :**
- `kubectl-who-can` : Vérifier les permissions RBAC
- `kubesec` : Analyser les manifestes de sécurité
- `kube-bench` : Vérifier la conformité CIS

**Pour le scanning :**
- `trivy` : Scanner de vulnérabilités pour les images
- `falco` : Détection d'anomalies runtime
- `OPA (Open Policy Agent)` : Politiques de sécurité as code

### Commandes de diagnostic utiles

```bash
# Vérifier les permissions d'un utilisateur
kubectl auth can-i --list

# Auditer les pods avec des privilèges élevés
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.securityContext.privileged}{"\n"}{end}'

# Lister les secrets dans tous les namespaces
kubectl get secrets --all-namespaces

# Vérifier les Network Policies actives
kubectl get networkpolicies --all-namespaces
```

## Checklist de sécurité initiale

Avant de commencer à déployer des applications dans votre lab, assurez-vous d'avoir :

- [ ] **Accès au cluster sécurisé**
  - [ ] kubectl configuré avec des certificats appropriés
  - [ ] Pas d'accès root direct non nécessaire
  - [ ] Fichier kubeconfig protégé (permissions 600)

- [ ] **Isolation de base configurée**
  - [ ] Namespaces créés pour séparer les workloads
  - [ ] RBAC activé sur le cluster
  - [ ] ServiceAccounts par défaut restreints

- [ ] **Réseau sécurisé**
  - [ ] Pas de services exposés inutilement
  - [ ] Firewall configuré sur l'hôte
  - [ ] Ingress avec HTTPS configuré

- [ ] **Gestion des secrets préparée**
  - [ ] Stratégie pour stocker les secrets
  - [ ] Processus de rotation défini
  - [ ] Backup des secrets critiques

## Points clés à retenir

La sécurité dans Kubernetes est un voyage, pas une destination. Même dans un environnement de lab :

1. **Commencez simple** : Implémentez d'abord les contrôles de base avant de complexifier
2. **Défense en profondeur** : Utilisez plusieurs couches de sécurité
3. **Principe du moindre privilège** : N'accordez que les permissions strictement nécessaires
4. **Automatisez** : Les processus manuels sont sources d'erreurs
5. **Restez informé** : La sécurité Kubernetes évolue rapidement

## Ce que nous allons couvrir

Dans les sections suivantes, nous allons implémenter concrètement chaque aspect de la sécurité :

- **11.1** RBAC : Contrôler qui peut faire quoi dans le cluster
- **11.2** Network Policies : Isoler les communications entre pods
- **11.3** Pod Security Standards : Définir les contraintes de sécurité des pods
- **11.4** Scan de vulnérabilités : Détecter les failles dans vos images
- **11.5** Secrets management : Gérer les données sensibles de manière sécurisée
- **11.6** Audit logging : Tracer toutes les actions dans le cluster
- **11.7** Bonnes pratiques : Synthèse et recommandations

Chaque section inclura des exemples pratiques adaptés à votre lab MicroK8s, vous permettant d'expérimenter en toute sécurité avec ces concepts essentiels.

---

*Note : La sécurité est un domaine en constante évolution. Cette documentation reflète les meilleures pratiques actuelles, mais il est important de rester informé des dernières vulnérabilités et recommandations de la communauté Kubernetes.*

⏭️
