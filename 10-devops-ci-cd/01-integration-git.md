🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1 Intégration avec Git

## Introduction : Pourquoi Git est essentiel pour Kubernetes

Git est le système de contrôle de version le plus utilisé au monde, et son intégration avec Kubernetes représente la pierre angulaire de toute approche DevOps moderne. Dans votre lab MicroK8s, Git servira de colonne vertébrale pour gérer, versionner et déployer vos applications et configurations.

## Comprendre le rôle de Git dans l'écosystème Kubernetes

### Le concept de "Configuration as Code"

Dans l'univers Kubernetes, tout est décrit par des fichiers YAML : vos déploiements, services, ingress, configmaps, secrets, etc. Ces fichiers constituent le code source de votre infrastructure. Git vous permet de :

- **Versionner** chaque modification de votre infrastructure
- **Collaborer** avec d'autres développeurs sur les mêmes configurations
- **Revenir en arrière** en cas de problème
- **Documenter** les changements via les messages de commit
- **Automatiser** les déploiements basés sur les modifications

### La différence avec une approche traditionnelle

Sans Git, vous pourriez modifier directement vos ressources Kubernetes via `kubectl edit` ou l'interface dashboard. Cette approche présente plusieurs problèmes majeurs :

- Aucune traçabilité des modifications
- Impossible de savoir qui a changé quoi et pourquoi
- Difficile de reproduire l'environnement ailleurs
- Pas de mécanisme de rollback simple
- Collaboration complexe entre plusieurs personnes

Avec Git, chaque modification est documentée, versionnée et peut être auditée. Votre infrastructure devient reproductible et prévisible.

## Structure d'un repository Git pour Kubernetes

### Organisation basique recommandée

Pour un lab MicroK8s personnel, voici une structure de repository claire et efficace :

```
mon-lab-k8s/
├── README.md                 # Documentation du projet
├── .gitignore               # Fichiers à ignorer
├── environments/            # Configurations par environnement
│   ├── dev/                # Environnement de développement
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   ├── staging/            # Environnement de staging
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   └── prod/               # Environnement de production
│       ├── namespace.yaml
│       └── kustomization.yaml
├── applications/           # Applications déployées
│   ├── app1/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── configmap.yaml
│   └── app2/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── infrastructure/         # Composants d'infrastructure
│   ├── monitoring/
│   │   ├── prometheus/
│   │   └── grafana/
│   ├── ingress-nginx/
│   └── cert-manager/
├── scripts/               # Scripts d'automatisation
│   ├── deploy.sh
│   └── rollback.sh
└── docs/                  # Documentation additionnelle
    ├── architecture.md
    └── runbooks/
```

### Le fichier .gitignore essentiel

Certains fichiers ne doivent jamais être versionnés dans Git. Créez un fichier `.gitignore` adapté :

```gitignore
# Secrets et données sensibles
*.key
*.pem
*.crt
secrets.yaml
.env
.env.local

# Fichiers temporaires
*.tmp
*.backup
*.swp
.DS_Store

# Fichiers générés
*.generated.yaml
helm-charts/*.tgz

# Kubeconfig local
kubeconfig
.kube/

# IDE
.vscode/
.idea/
*.iml

# Logs
*.log
logs/
```

## Workflow Git pour Kubernetes

### Le cycle de vie d'une modification

Voici le workflow typique pour modifier votre infrastructure Kubernetes via Git :

#### 1. Cloner ou mettre à jour le repository

```bash
# Premier clone
git clone https://github.com/moncompte/mon-lab-k8s.git
cd mon-lab-k8s

# Ou mise à jour d'un repo existant
git pull origin main
```

#### 2. Créer une branche pour vos modifications

Les branches permettent d'isoler vos changements. Adoptez une convention de nommage claire :

```bash
# Pour une nouvelle fonctionnalité
git checkout -b feature/ajouter-prometheus

# Pour une correction
git checkout -b fix/corriger-ingress-app1

# Pour une mise à jour
git checkout -b update/upgrade-nginx-1.21
```

#### 3. Effectuer vos modifications

Modifiez vos fichiers YAML avec votre éditeur préféré. Par exemple, pour ajouter une nouvelle application :

```yaml
# applications/app3/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
    spec:
      containers:
      - name: app3
        image: myregistry/app3:v1.0.0
        ports:
        - containerPort: 8080
```

#### 4. Valider localement

Avant de commiter, validez toujours vos manifestes :

```bash
# Validation syntaxique
kubectl apply --dry-run=client -f applications/app3/deployment.yaml

# Ou validation sur le serveur (plus complet)
kubectl apply --dry-run=server -f applications/app3/deployment.yaml
```

#### 5. Commiter vos changements

Utilisez des messages de commit descriptifs suivant une convention :

```bash
# Ajouter les fichiers modifiés
git add applications/app3/

# Commiter avec un message descriptif
git commit -m "feat: Ajouter l'application app3 avec 2 replicas

- Création du deployment avec l'image v1.0.0
- Configuration de 2 replicas pour la haute disponibilité
- Port 8080 exposé

Refs: #123"
```

### Conventions de messages de commit

Adoptez une convention comme Conventional Commits pour structurer vos messages :

- **feat:** Nouvelle fonctionnalité
- **fix:** Correction de bug
- **docs:** Documentation uniquement
- **style:** Formatage, pas de changement de code
- **refactor:** Refactoring du code
- **perf:** Amélioration des performances
- **test:** Ajout de tests
- **chore:** Maintenance, dépendances

#### 6. Pousser vers le repository distant

```bash
git push origin feature/ajouter-prometheus
```

#### 7. Créer une Pull Request / Merge Request

Même dans un lab personnel, utilisez les PR/MR pour :
- Réviser vos propres changements
- Exécuter des tests automatiques
- Documenter les décisions
- Garder un historique clair

## Stratégies de branches pour Kubernetes

### Stratégie simple : GitHub Flow

Pour un lab personnel, le GitHub Flow est idéal :

1. **main** : Branche principale, toujours déployable
2. **feature branches** : Branches temporaires pour les modifications
3. **Merge** : Fusion dans main après validation

```
main ──────●──────●──────●──────●
            \    /  \    /
feature/x    ●──●    ●──●
```

### Stratégie avancée : GitFlow adapté

Pour des environnements multiples (dev, staging, prod) :

```
main     ──────────●──────────●────── (production)
                  /          /
release  ────●───●──────●───●──────── (staging)
            /          /
develop  ──●──●──●──●──●──●──●──────── (développement)
          /    \  /    \
feature  ●──●   ●●      ●──●
```

### Stratégie GitOps : Environment Branches

Une branche par environnement, idéale pour GitOps :

```
prod     ──●──────●──────●──────●──── → Cluster Prod
           ↑      ↑      ↑
staging  ──●──●───●──●───●──●──────── → Cluster Staging
           ↑  ↑      ↑      ↑
dev      ──●──●──●───●──●───●──●──── → Cluster Dev
```

## Gestion des secrets avec Git

### Le problème des données sensibles

Les secrets (mots de passe, clés API, certificats) ne doivent JAMAIS être committés en clair dans Git. Voici les approches recommandées :

### Approche 1 : Sealed Secrets

Sealed Secrets chiffre vos secrets avant de les stocker dans Git :

```yaml
# secret-clair.yaml (NE PAS COMMITTER)
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  password: bXlzcWxwYXNzd29yZA==  # base64 de "mysqlpassword"
```

Après chiffrement avec Sealed Secrets :

```yaml
# sealed-secret.yaml (PEUT ÊTRE COMMITTÉ)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysql-secret
spec:
  encryptedData:
    password: AgXZOs8p2H4W1t7...  # Chiffré, safe pour Git
```

### Approche 2 : Référencer des secrets externes

Stockez uniquement des références dans Git :

```yaml
# configmap.yaml (committé dans Git)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_secret_name: "mysql-credentials"  # Référence seulement
```

### Approche 3 : Variables d'environnement CI/CD

Les secrets sont injectés lors du déploiement :

```yaml
# deployment-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: DB_PASSWORD
          value: ${DB_PASSWORD}  # Remplacé par le CI/CD
```

## Synchronisation Git-Kubernetes

### Méthode manuelle

La méthode la plus simple pour appliquer vos configurations :

```bash
# Après avoir pull les dernières modifications
git pull origin main

# Appliquer toutes les configurations d'un dossier
kubectl apply -f applications/app1/

# Ou appliquer récursivement
kubectl apply -R -f applications/
```

### Méthode semi-automatique avec scripts

Créez des scripts pour standardiser les déploiements :

```bash
#!/bin/bash
# scripts/deploy.sh

set -e  # Arrêter en cas d'erreur

ENVIRONMENT=${1:-dev}
APP_NAME=${2:-all}

echo "Déploiement de $APP_NAME vers $ENVIRONMENT"

# Mise à jour du repo
git pull origin main

# Validation des manifestes
echo "Validation des manifestes..."
if [ "$APP_NAME" = "all" ]; then
    kubectl apply --dry-run=server -R -f applications/
else
    kubectl apply --dry-run=server -f applications/$APP_NAME/
fi

# Application
echo "Application des manifestes..."
if [ "$APP_NAME" = "all" ]; then
    kubectl apply -R -f applications/
else
    kubectl apply -f applications/$APP_NAME/
fi

echo "Déploiement terminé avec succès!"
```

### Méthode GitOps automatique

Avec ArgoCD ou Flux, le cluster se synchronise automatiquement avec Git :

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mon-lab
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/moncompte/mon-lab-k8s
    targetRevision: main
    path: applications
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Bonnes pratiques Git pour Kubernetes

### 1. Un commit = Une modification logique

Ne mélangez pas plusieurs changements dans un même commit :

```bash
# ❌ Mauvais
git commit -m "Mise à jour de plusieurs trucs"

# ✅ Bon
git commit -m "feat(app1): Augmenter les replicas à 3 pour la charge"
git commit -m "fix(ingress): Corriger le path pour app2"
```

### 2. Utiliser des tags pour les versions

Marquez les versions importantes :

```bash
# Créer un tag
git tag -a v1.0.0 -m "Version 1.0.0 - Première release stable"

# Pousser les tags
git push origin --tags

# Déployer une version spécifique
git checkout v1.0.0
kubectl apply -R -f applications/
```

### 3. Documenter dans les commits ET le code

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  annotations:
    # Documentation importante directement dans le YAML
    description: "Application principale de gestion des commandes"
    owner: "equipe-backend"
    documentation: "https://wiki.internal/app1"
spec:
  replicas: 3  # Augmenté de 2 à 3 suite aux pics de charge (ticket #456)
```

### 4. Révision de code systématique

Même seul, utilisez les Pull Requests pour :

- Relire vos modifications
- Vérifier les différences (diff)
- Exécuter des tests automatiques
- Documenter les décisions

### 5. Backup et redondance

Git distribué offre une redondance naturelle :

```bash
# Ajouter plusieurs remotes
git remote add github https://github.com/moncompte/mon-lab-k8s.git
git remote add gitlab https://gitlab.com/moncompte/mon-lab-k8s.git

# Pousser vers plusieurs remotes
git push github main
git push gitlab main
```

## Intégration avec les outils Kubernetes

### kubectl et Git

Créez des alias pour faciliter l'intégration :

```bash
# Dans ~/.bashrc ou ~/.zshrc
alias kgit='kubectl apply -f'
alias kgitdry='kubectl apply --dry-run=server -f'
alias kgitdiff='kubectl diff -f'

# Utilisation
kgitdry applications/app1/deployment.yaml
kgitdiff applications/app1/  # Voir les différences avec le cluster
```

### Kustomize et Git

Kustomize permet de gérer des variations sans dupliquer :

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

patchesStrategicMerge:
  - increase-replicas.yaml
```

### Helm et Git

Versionnez vos values Helm :

```yaml
# helm-values/app1/values-dev.yaml
replicaCount: 1
image:
  tag: latest

# helm-values/app1/values-prod.yaml
replicaCount: 3
image:
  tag: v1.0.0
```

## Troubleshooting : Problèmes courants

### Conflit lors du merge

Quand deux branches modifient le même fichier :

```bash
# Récupérer les modifications
git pull origin main

# En cas de conflit
git status  # Voir les fichiers en conflit

# Éditer manuellement les fichiers pour résoudre
vim applications/app1/deployment.yaml

# Marquer comme résolu
git add applications/app1/deployment.yaml
git commit -m "fix: Résoudre conflit sur deployment.yaml"
```

### Rollback d'une mauvaise configuration

```bash
# Voir l'historique
git log --oneline

# Revenir à un commit précédent
git revert HEAD  # Annule le dernier commit
# ou
git reset --hard abc1234  # Revenir à un commit spécifique

# Réappliquer au cluster
kubectl apply -R -f applications/
```

### Synchronisation désynchronisée

Si le cluster ne correspond plus à Git :

```bash
# Forcer la resynchronisation complète
kubectl delete -R -f applications/
kubectl apply -R -f applications/

# Ou avec une approche plus douce
kubectl apply -R -f applications/ --force
```

## Monitoring et audit avec Git

### Tracer qui a fait quoi

```bash
# Voir l'historique d'un fichier
git log -p applications/app1/deployment.yaml

# Voir qui a modifié chaque ligne
git blame applications/app1/deployment.yaml

# Chercher dans l'historique
git log --grep="scaling"  # Tous les commits mentionnant "scaling"
```

### Intégrer Git avec votre monitoring

Ajoutez des annotations Kubernetes qui référencent Git :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  annotations:
    git.commit: "${GIT_COMMIT}"
    git.branch: "${GIT_BRANCH}"
    deployed.by: "${USER}"
    deployed.at: "${DATE}"
```

## Conclusion et prochaines étapes

L'intégration de Git avec votre lab MicroK8s transforme fondamentalement votre approche de la gestion d'infrastructure. Vous passez d'une gestion manuelle et opaque à une approche versionnée, traçable et collaborative.

Les bénéfices immédiats incluent :
- **Traçabilité complète** : Chaque changement est documenté
- **Rollback facile** : Retour arrière en quelques commandes
- **Collaboration** : Même seul, vous collaborez avec votre "futur vous"
- **Automatisation** : Base pour le CI/CD et GitOps
- **Documentation vivante** : Votre Git devient la documentation

La maîtrise de Git pour Kubernetes est une compétence fondamentale qui vous servira dans tous vos projets, du lab personnel aux déploiements d'entreprise. Les concepts appris ici s'appliquent directement aux environnements professionnels, où Git est universellement adopté pour la gestion d'infrastructure.

Dans la section suivante, nous explorerons comment transformer ce repository Git en véritable usine logicielle avec la mise en place d'un registry privé pour vos images Docker.

⏭️
