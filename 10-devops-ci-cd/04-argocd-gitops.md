🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 ArgoCD pour GitOps

## Introduction : La révolution GitOps

GitOps représente un changement de paradigme dans la gestion d'infrastructure. Au lieu de pousser des changements vers votre cluster (push), c'est le cluster qui tire automatiquement les configurations depuis Git (pull). ArgoCD est l'outil qui concrétise cette vision, transformant Git en source unique de vérité pour l'état de votre infrastructure Kubernetes.

## Comprendre GitOps en profondeur

### Le principe fondamental

Imaginez votre repository Git comme un plan d'architecte et ArgoCD comme le contremaître qui s'assure que la construction (votre cluster) correspond exactement au plan. Si quelqu'un modifie directement le bâtiment sans mettre à jour les plans, le contremaître détecte l'écart et corrige automatiquement la construction pour qu'elle corresponde aux plans officiels.

### Les quatre principes GitOps

**1. Déclaratif** : L'ensemble de votre système est décrit de manière déclarative. Vous décrivez l'état désiré, pas les étapes pour y arriver.

**2. Versionné et immutable** : L'état désiré est stocké dans Git, offrant un historique complet, immutable et auditable de tous les changements.

**3. Automatiquement appliqué** : Les changements approuvés sont automatiquement appliqués au système sans intervention manuelle.

**4. Continuellement réconcilié** : Des agents logiciels surveillent en permanence l'état actuel et le comparent à l'état désiré, corrigeant automatiquement toute dérive.

### Pourquoi ArgoCD ?

ArgoCD est devenu le standard de facto pour GitOps car il offre :

- **Interface web intuitive** : Visualisation graphique de vos applications
- **Multi-clusters** : Gestion de plusieurs clusters depuis une instance
- **Multi-tenancy** : Isolation entre équipes et projets
- **Sync automatique** : Réconciliation continue avec Git
- **Rollback facile** : Retour à n'importe quel état précédent
- **SSO/RBAC** : Sécurité enterprise-grade
- **Extensibilité** : Hooks, plugins, et personnalisation

## Architecture d'ArgoCD

### Composants principaux

**API Server** : Le cerveau d'ArgoCD, expose l'API REST/gRPC, gère l'authentification, et coordonne tous les composants.

**Repository Server** : Responsable de cloner les repos Git et de générer les manifestes Kubernetes. Il supporte Helm, Kustomize, Jsonnet, et les manifestes YAML bruts.

**Application Controller** : Surveille continuellement les applications, compare l'état actuel avec l'état désiré dans Git, et prend les actions nécessaires pour les synchroniser.

**Redis** : Cache pour améliorer les performances et stocker l'état temporaire.

**Dex** : Serveur OpenID Connect pour l'authentification SSO (optionnel).

### Flux de données

```
Git Repository → Repository Server → Application Controller → Kubernetes Cluster
                                            ↓
                                    Monitoring & Reconciliation
                                            ↓
                                      UI/CLI/API
```

## Installation d'ArgoCD sur MicroK8s

### Installation standard avec kubectl

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Attendre que tous les pods soient prêts
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Vérifier l'installation
kubectl get pods -n argocd
```

### Installation personnalisée avec Helm

```bash
# Ajouter le repository Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Créer un fichier de values personnalisées
cat > argocd-values.yaml <<EOF
## Configuration ArgoCD pour MicroK8s Lab
global:
  domain: argocd.monlab.local

## Configuration du serveur
server:
  # Interface web
  extraArgs:
    - --insecure  # HTTP pour le lab (HTTPS en production)

  # Ingress configuration
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argocd.monlab.local
    paths:
      - /
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.monlab.local

  # Métriques Prometheus
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

## Repository Server - Support des outils
repoServer:
  volumes:
    - name: custom-tools
      emptyDir: {}
  volumeMounts:
    - name: custom-tools
      mountPath: /usr/local/bin
  initContainers:
    - name: download-tools
      image: alpine:latest
      command: [sh, -c]
      args:
        - |
          wget -O /usr/local/bin/kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.0.0/kustomize_v5.0.0_linux_amd64.tar.gz
          tar -xzf /usr/local/bin/kustomize -C /usr/local/bin
          chmod +x /usr/local/bin/kustomize
          rm /usr/local/bin/kustomize.tar.gz
      volumeMounts:
        - name: custom-tools
          mountPath: /usr/local/bin

## Controller - Performance tuning
controller:
  replicas: 1
  metrics:
    enabled: true

## Redis - Configuration
redis:
  enabled: true

## Dex - Pour SSO (optionnel)
dex:
  enabled: false  # Activer pour SSO

## Notifications
notifications:
  enabled: true
  argocdUrl: https://argocd.monlab.local

  # Webhook pour Slack
  notifiers:
    service.slack: |
      token: $slack-token

  # Templates de notifications
  templates:
    template.app-deployed: |
      message: |
        {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} déployée avec succès.
        Révision: {{.app.status.sync.revision}}

  # Triggers
  triggers:
    trigger.on-deployed: |
      - when: app.status.operationState.phase in ['Succeeded']
        send: [app-deployed]
EOF

# Installer avec Helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd-values.yaml
```

### Configuration de l'accès

```bash
# Récupérer le mot de passe admin initial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Exposer l'interface web (développement)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Ou créer un Ingress
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: argocd.monlab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF
```

### Installation et configuration du CLI

```bash
# Télécharger ArgoCD CLI
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

chmod +x /usr/local/bin/argocd

# Se connecter au serveur
argocd login argocd.monlab.local:443 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# Changer le mot de passe admin
argocd account update-password

# Lister les applications
argocd app list
```

## Configuration d'applications avec ArgoCD

### Structure d'un repository GitOps

```
gitops-repo/
├── README.md
├── apps/                      # Définitions des applications ArgoCD
│   ├── dev/
│   │   ├── app1.yaml
│   │   ├── app2.yaml
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
├── infrastructure/            # Components d'infrastructure
│   ├── ingress-nginx/
│   ├── cert-manager/
│   ├── prometheus/
│   └── grafana/
├── clusters/                  # Configuration par cluster
│   ├── dev-cluster/
│   │   ├── apps.yaml         # App of Apps
│   │   └── infrastructure.yaml
│   └── prod-cluster/
│       └── ...
└── overlays/                  # Kustomize overlays
    ├── dev/
    ├── staging/
    └── production/
```

### Création d'une application simple

```yaml
# apps/dev/hello-world.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello-world
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # Project auquel l'app appartient
  project: default

  # Source Git
  source:
    repoURL: https://github.com/monlab/hello-world
    targetRevision: HEAD
    path: kubernetes/

  # Destination (cluster et namespace)
  destination:
    server: https://kubernetes.default.svc
    namespace: hello-world

  # Politique de synchronisation
  syncPolicy:
    automated:
      prune: true        # Supprimer les ressources qui ne sont plus dans Git
      selfHeal: true     # Corriger automatiquement les dérives
      allowEmpty: false  # Ne pas supprimer toutes les ressources si Git est vide
    syncOptions:
      - CreateNamespace=true  # Créer le namespace si nécessaire
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Application avec Helm

```yaml
# apps/dev/wordpress-helm.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wordpress
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: wordpress
    targetRevision: 18.0.0
    helm:
      releaseName: wordpress
      values: |
        # Values Helm personnalisées
        wordpressUsername: admin
        wordpressEmail: admin@monlab.local
        wordpressBlogName: "Mon Lab Blog"

        service:
          type: ClusterIP

        ingress:
          enabled: true
          hostname: blog.monlab.local
          ingressClassName: nginx
          tls: true

        mariadb:
          enabled: true
          auth:
            database: wordpress
            username: wordpress
          primary:
            persistence:
              enabled: true
              size: 8Gi

        persistence:
          enabled: true
          size: 10Gi

  destination:
    server: https://kubernetes.default.svc
    namespace: wordpress

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application avec Kustomize

```yaml
# apps/dev/myapp-kustomize.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-kustomize
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/monlab/myapp
    targetRevision: main
    path: deploy/overlays/dev

    # Configuration Kustomize
    kustomize:
      namePrefix: dev-
      nameSuffix: -v1
      commonLabels:
        environment: dev
        managed-by: argocd
      images:
        - myapp=registry.monlab.local:5000/myapp:latest
      replicas:
        - name: myapp-deployment
          count: 2

  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Pattern "App of Apps"

Le pattern "App of Apps" permet de gérer toutes vos applications avec une seule application ArgoCD racine :

```yaml
# clusters/dev-cluster/apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-apps
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/monlab/gitops
    targetRevision: HEAD
    path: apps/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Cette application va automatiquement découvrir et déployer toutes les applications définies dans `apps/dev/`.

## Projects ArgoCD

Les projects permettent de gérer les permissions et restrictions :

```yaml
# argocd-projects/development.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: development
  namespace: argocd
spec:
  description: Projet pour l'environnement de développement

  # Sources Git autorisées
  sourceRepos:
    - 'https://github.com/monlab/*'
    - 'https://gitlab.monlab.local/*'

  # Destinations autorisées
  destinations:
    - namespace: 'dev-*'
      server: https://kubernetes.default.svc
    - namespace: 'development'
      server: https://kubernetes.default.svc

  # Ressources Kubernetes autorisées
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'

  # Ressources interdites
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: NetworkPolicy

  # Roles RBAC
  roles:
    - name: developers
      policies:
        - p, proj:development:developers, applications, get, development/*, allow
        - p, proj:development:developers, applications, sync, development/*, allow
      groups:
        - monlab:developers

    - name: admins
      policies:
        - p, proj:development:admins, applications, *, development/*, allow
        - p, proj:development:admins, repositories, *, *, allow
      groups:
        - monlab:admins
```

## Gestion des secrets avec ArgoCD

### Sealed Secrets Integration

```yaml
# Installation de Sealed Secrets Controller
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-controller
  namespace: argocd
spec:
  project: infrastructure
  source:
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    chart: sealed-secrets
    targetRevision: 2.13.0
    helm:
      values: |
        fullnameOverride: sealed-secrets-controller
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Utilisation dans vos manifestes :

```yaml
# secret-sealed.yaml (dans Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  encryptedData:
    db-password: AgBvA8N2H4W1t7... # Chiffré, safe pour Git
    api-key: AgXZOs8p2H4W1t7...
```

### ArgoCD Vault Plugin

```yaml
# Configuration du plugin Vault
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-vault-plugin
  namespace: argocd
data:
  plugin.yaml: |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin
    spec:
      version: v1.0
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
      discover:
        find:
          glob: "**/*.yaml"
```

Utilisation dans les manifestes :

```yaml
# deployment.yaml avec placeholders Vault
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
          value: <path:secret/data/myapp#db-password>
        - name: API_KEY
          value: <path:secret/data/myapp#api-key>
```

### External Secrets Operator

```yaml
# Installation d'External Secrets
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
spec:
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: 0.9.0
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
```

## Stratégies de synchronisation

### Sync manuel avec hooks

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-manual
spec:
  syncPolicy:
    # Pas d'automation
    syncOptions:
      - ApplyOutOfSyncOnly=true  # N'appliquer que les ressources out-of-sync
      - PrunePropagationPolicy=foreground  # Stratégie de suppression
      - RespectIgnoreDifferences=true

  # Hooks de synchronisation
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # Ignorer les changements de replicas (HPA)
```

### Progressive Sync avec Waves

```yaml
# Infrastructure d'abord, puis apps
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
  annotations:
    argocd.argoproj.io/sync-wave: "-2"  # Déployé en premier
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # Puis la DB
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Puis le backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Enfin le frontend
```

### Hooks de synchronisation

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync  # Avant la sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate/migrate
        command: ["migrate", "-path", "/migrations", "-database", "$DATABASE_URL", "up"]
      restartPolicy: Never
---
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-tests
  annotations:
    argocd.argoproj.io/hook: PostSync  # Après la sync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: test
        image: myapp:test
        command: ["npm", "run", "test:smoke"]
      restartPolicy: Never
```

## Multi-cluster avec ArgoCD

### Ajout d'un cluster externe

```bash
# Depuis le cluster où ArgoCD est installé
# Ajouter le contexte du cluster distant
argocd cluster add microk8s-prod \
  --name production-cluster \
  --kubeconfig /path/to/kubeconfig

# Lister les clusters
argocd cluster list

# Vérifier la connexion
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=cluster
```

### ApplicationSet pour multi-cluster

```yaml
# applicationset-multi-cluster.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-apps
  namespace: argocd
spec:
  generators:
  - clusters: {}  # Génère une app par cluster

  template:
    metadata:
      name: '{{name}}-apps'
    spec:
      project: default
      source:
        repoURL: https://github.com/monlab/gitops
        targetRevision: HEAD
        path: 'clusters/{{name}}'
      destination:
        server: '{{server}}'
        namespace: argocd
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### ApplicationSet avec Matrix Generator

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: matrix-deployment
spec:
  generators:
  - matrix:
      generators:
      - clusters:
          selector:
            matchLabels:
              environment: production
      - list:
          elements:
          - app: frontend
            version: v1.0.0
          - app: backend
            version: v2.0.0

  template:
    metadata:
      name: '{{name}}-{{app}}'
    spec:
      source:
        repoURL: https://github.com/monlab/{{app}}
        targetRevision: '{{version}}'
        path: kubernetes/
      destination:
        server: '{{server}}'
        namespace: '{{app}}'
```

## Monitoring et observabilité

### Métriques Prometheus

```yaml
# ServiceMonitor pour ArgoCD
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Dashboard Grafana pour ArgoCD

```json
{
  "dashboard": {
    "title": "ArgoCD Operations",
    "panels": [
      {
        "title": "Applications Health",
        "targets": [{
          "expr": "sum by (health_status) (argocd_app_info)"
        }]
      },
      {
        "title": "Sync Operations",
        "targets": [{
          "expr": "rate(argocd_app_sync_total[5m])"
        }]
      },
      {
        "title": "Reconciliation Duration",
        "targets": [{
          "expr": "histogram_quantile(0.95, argocd_app_reconcile_duration_seconds_bucket)"
        }]
      },
      {
        "title": "Git Requests Rate",
        "targets": [{
          "expr": "rate(argocd_git_request_total[5m])"
        }]
      }
    ]
  }
}
```

### Notifications

```yaml
# Configuration des notifications Slack
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token

  template.app-deployed: |
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application *{{.app.metadata.name}}* déployée avec succès
      Sync Status: {{.app.status.sync.status}}
      Health: {{.app.status.health.status}}
      <{{.context.argocdUrl}}/applications/{{.app.metadata.name}}|Voir dans ArgoCD>

  template.app-health-degraded: |
    message: |
      {{if eq .serviceType "slack"}}:exclamation:{{end}} Application *{{.app.metadata.name}}* dégradée
      Health: {{.app.status.health.status}}
      Message: {{.app.status.health.message}}

  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
```

## Rollback et disaster recovery

### Rollback manuel

```bash
# Via CLI
argocd app rollback myapp 1.0.0  # Vers une version spécifique
argocd app rollback myapp --revision 3  # Vers une révision

# Via l'interface web
# Applications → myapp → History → Rollback

# Via kubectl
kubectl patch application myapp -n argocd --type merge -p '
{
  "spec": {
    "source": {
      "targetRevision": "v1.0.0"
    }
  }
}'
```

### Backup et restauration

```bash
# Backup des applications ArgoCD
kubectl get applications -n argocd -o yaml > argocd-apps-backup.yaml
kubectl get appprojects -n argocd -o yaml > argocd-projects-backup.yaml

# Backup avec argocd-util
argocd-util export > argocd-backup.yaml

# Restauration
kubectl apply -f argocd-apps-backup.yaml
kubectl apply -f argocd-projects-backup.yaml

# Ou avec argocd-util
argocd-util import < argocd-backup.yaml
```

### Plan de disaster recovery

```yaml
# dr-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-backup
  namespace: argocd
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              DATE=$(date +%Y%m%d-%H%M%S)
              kubectl get applications,appprojects,secrets -n argocd -o yaml > /backup/argocd-$DATE.yaml
              # Upload vers S3 ou autre stockage
              aws s3 cp /backup/argocd-$DATE.yaml s3://backup-bucket/argocd/
              # Nettoyer les vieux backups (garder 30 jours)
              find /backup -name "argocd-*.yaml" -mtime +30 -delete
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: argocd-backup-pvc
          restartPolicy: OnFailure
```

## Sécurité et RBAC

### Configuration RBAC détaillée

```yaml
# argocd-rbac-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Admins - Accès complet
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    p, role:admin, projects, *, *, allow
    g, argocd-admins, role:admin

    # Developers - Accès aux apps dev/staging
    p, role:developer, applications, get, dev/*, allow
    p, role:developer, applications, sync, dev/*, allow
    p, role:developer, applications, action/*, dev/*, allow
    p, role:developer, applications, get, staging/*, allow
    p, role:developer, applications, sync, staging/*, allow
    p, role:developer, logs, get, dev/*, allow
    p, role:developer, logs, get, staging/*, allow
    g, developers, role:developer

    # Ops - Gestion production
    p, role:ops, applications, *, production/*, allow
    p, role:ops, clusters, get, *, allow
    p, role:ops, projects, get, *, allow
    g, operations, role:ops

    # Readonly - Lecture seule
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, projects, get, *, allow
    g, guests, role:readonly
```

### SSO avec Dex et OAuth2

```yaml
# argocd-cm avec configuration Dex
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.monlab.local

  dex.config: |
    connectors:
    # GitHub OAuth
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $dex.github.clientId
        clientSecret: $dex.github.clientSecret
        orgs:
        - name: monlab
          teams:
          - developers
          - operations

    # LDAP
    - type: ldap
      id: ldap
      name: LDAP
      config:
        host: ldap.monlab.local:389
        insecureNoSSL: true
        bindDN: cn=admin,dc=monlab,dc=local
        bindPW: $dex.ldap.bindPW
        userSearch:
          baseDN: ou=users,dc=monlab,dc=local
          filter: "(objectClass=person)"
          username: uid
          idAttr: uid
          emailAttr: mail
          nameAttr: cn
        groupSearch:
          baseDN: ou=groups,dc=monlab,dc=local
          filter: "(objectClass=groupOfNames)"
          userAttr: DN
          groupAttr: member
          nameAttr: cn

    # OIDC (Keycloak, Okta, etc.)
    - type: oidc
      id: oidc
      name: Keycloak
      config:
        issuer: https://keycloak.monlab.local/auth/realms/master
        clientID: $dex.oidc.clientId
        clientSecret: $dex.oidc.clientSecret
        redirectURI: https://argocd.monlab.local/api/dex/callback
        scopes:
        - openid
        - profile
        - email
        - groups
        userIDKey: preferred_username
        userNameKey: name

  # Configuration OIDC directe (sans Dex)
  oidc.config: |
    name: Keycloak
    issuer: https://keycloak.monlab.local/auth/realms/master
    clientId: argocd
    clientSecret: $oidc.keycloak.clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

### Configuration des groupes et permissions

```yaml
# argocd-rbac-cm pour SSO
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly

  # Mappage des groupes SSO vers les rôles ArgoCD
  policy.csv: |
    # GitHub teams
    g, monlab:developers, role:developer
    g, monlab:operations, role:admin

    # LDAP groups
    g, cn=developers,ou=groups,dc=monlab,dc=local, role:developer
    g, cn=admins,ou=groups,dc=monlab,dc=local, role:admin

    # Keycloak groups
    g, /developers, role:developer
    g, /admins, role:admin

    # Permissions par rôle
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, dev/*, allow
    p, role:developer, applications, sync, staging/*, allow
    p, role:developer, applications, action/*, dev/*, allow
    p, role:developer, logs, get, */*, allow
    p, role:developer, exec, create, dev/*, allow

    p, role:admin, *, *, */*, allow

  # Scopes pour les tokens API
  scopes: '[groups, email]'
```

## Optimisation des performances

### Configuration du cache Redis

```yaml
# redis-ha pour haute disponibilité
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-redis-ha-configmap
  namespace: argocd
data:
  redis.conf: |
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    save 60 10000
    appendonly yes
    appendfsync everysec

  sentinel.conf: |
    sentinel down-after-milliseconds mymaster 5000
    sentinel failover-timeout mymaster 10000
    sentinel parallel-syncs mymaster 1
```

### Tuning du controller

```yaml
# argocd-cmd-params-cm pour optimisation
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  # Nombre de workers pour le traitement parallèle
  controller.status.processors: "50"
  controller.operation.processors: "25"

  # Réduire la fréquence de réconciliation (défaut: 3m)
  timeout.reconciliation: "180s"

  # Cache des manifestes
  controller.repo.server.timeout.seconds: "60"
  reposerver.parallelism.limit: "10"

  # Optimisation réseau
  controller.k8sclient.retry.max: "5"
  controller.k8sclient.retry.base.backoff: "1s"

  # Garbage collection
  application.instanceLabelKey: argocd.argoproj.io/instance
  server.disable.auth: "false"

  # Logging
  controller.log.level: "info"
  server.log.level: "info"
  reposerver.log.level: "info"
```

### Sharding pour grandes installations

```yaml
# Sharding des applications entre plusieurs controllers
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  controller.sharding.algorithm: "hash"  # ou "round-robin"
  controller.sharding.replicas: "3"

---
# Deployment avec plusieurs replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-application-controller
  namespace: argocd
spec:
  replicas: 3  # Nombre de shards
  template:
    spec:
      containers:
      - name: argocd-application-controller
        env:
        - name: ARGOCD_CONTROLLER_REPLICAS
          value: "3"
        - name: ARGOCD_CONTROLLER_SHARD
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['argocd.argoproj.io/controller-shard']
```

## Patterns avancés GitOps

### Progressive Delivery avec Flagger

```yaml
# Installation de Flagger pour canary deployments
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flagger
  namespace: argocd
spec:
  source:
    repoURL: https://flagger.app
    chart: flagger
    targetRevision: 1.35.0
    helm:
      values: |
        prometheus:
          install: true
        meshProvider: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: flagger-system

---
# Canary deployment avec Flagger
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
    targetPort: 8080
  analysis:
    interval: 1m
    threshold: 10
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
  webhooks:
    - name: acceptance-test
      url: http://flagger-loadtester.test/
      timeout: 30s
      metadata:
        type: bash
        cmd: "curl -sd 'test' http://myapp-canary.production:80/health"
```

### Blue-Green avec Argo Rollouts

```yaml
# Installation d'Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
spec:
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: 2.35.0
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts

---
# Rollout Blue-Green
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
  namespace: production
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-preview
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-active
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: registry.monlab.local:5000/myapp:latest
```

### Multi-tenancy avec ApplicationSets

```yaml
# ApplicationSet pour multi-tenant
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/monlab/tenants
      revision: HEAD
      directories:
      - path: tenants/*

  template:
    metadata:
      name: '{{path.basename}}-apps'
    spec:
      project: '{{path.basename}}'
      source:
        repoURL: https://github.com/monlab/tenants
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'tenant-{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

## Intégration avec les outils DevOps

### Pipeline CI/CD avec ArgoCD

```yaml
# GitLab CI integration
deploy:
  stage: deploy
  script:
    - |
      # Mettre à jour l'image dans le repo GitOps
      git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.monlab.local/gitops/apps.git
      cd apps

      # Utiliser yq pour mettre à jour l'image
      yq eval -i '.spec.source.helm.values = "image.tag: '${CI_COMMIT_SHA}'"' \
        apps/production/myapp.yaml

      # Commit et push
      git config user.email "ci@monlab.local"
      git config user.name "GitLab CI"
      git add .
      git commit -m "Update myapp to ${CI_COMMIT_SHA}"
      git push origin main

      # Optionnel: Forcer la sync ArgoCD
      argocd app sync myapp --force
      argocd app wait myapp --health --timeout 300
```

### Webhook pour auto-sync

```yaml
# Configuration webhook GitLab/GitHub
apiVersion: v1
kind: Secret
metadata:
  name: argocd-webhook-secret
  namespace: argocd
type: Opaque
data:
  # Secret partagé avec GitLab/GitHub
  webhook.gitlab.secret: <base64-encoded-secret>
  webhook.github.secret: <base64-encoded-secret>

---
# Ingress pour webhooks
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-webhook
  namespace: argocd
spec:
  rules:
  - host: argocd.monlab.local
    http:
      paths:
      - path: /api/webhook
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

### Intégration avec Terraform

```hcl
# terraform/argocd-app.tf
resource "kubernetes_manifest" "argocd_app" {
  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "Application"
    metadata = {
      name      = "terraform-managed-app"
      namespace = "argocd"
    }
    spec = {
      project = "default"
      source = {
        repoURL        = var.git_repo
        targetRevision = var.git_branch
        path           = var.app_path
      }
      destination = {
        server    = "https://kubernetes.default.svc"
        namespace = var.target_namespace
      }
      syncPolicy = {
        automated = {
          prune    = true
          selfHeal = true
        }
      }
    }
  }
}
```

## Troubleshooting ArgoCD

### Problèmes de synchronisation

```bash
# Application stuck en "Syncing"
argocd app get myapp --refresh
argocd app terminate-op myapp

# Forcer la resynchronisation
argocd app sync myapp --force --prune

# Voir les événements détaillés
kubectl describe application myapp -n argocd

# Logs du controller
kubectl logs -n argocd deployment/argocd-application-controller -f

# Reset de l'application
argocd app delete myapp --cascade=false
kubectl delete application myapp -n argocd
# Puis recréer l'application
```

### Problèmes de performance

```bash
# Vérifier les métriques du controller
kubectl top pods -n argocd

# Augmenter les ressources
kubectl patch deployment argocd-application-controller -n argocd -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "argocd-application-controller",
          "resources": {
            "requests": {"memory": "512Mi", "cpu": "500m"},
            "limits": {"memory": "2Gi", "cpu": "2"}
          }
        }]
      }
    }
  }
}'

# Nettoyer le cache Redis
kubectl exec -it argocd-redis-master-0 -n argocd -- redis-cli FLUSHALL

# Réduire la fréquence de polling Git
kubectl patch configmap argocd-cm -n argocd -p '
{
  "data": {
    "timeout.reconciliation": "300s"
  }
}'
```

### Problèmes de certificats

```bash
# Ignorer les certificats auto-signés pour les repos Git
argocd repo add https://git.monlab.local/myrepo.git \
  --insecure-skip-server-verification

# Ou ajouter le certificat CA
kubectl create configmap argocd-tls-certs-cm \
  --from-file=git.monlab.local=/path/to/ca.crt \
  -n argocd

# Pour les registries Docker
kubectl patch configmap argocd-cm -n argocd -p '
{
  "data": {
    "registries": "registry.monlab.local: {insecure: true}"
  }
}'
```

### Debugging avancé

```bash
# Activer le mode debug
kubectl patch configmap argocd-cmd-params-cm -n argocd -p '
{
  "data": {
    "controller.log.level": "debug",
    "server.log.level": "debug",
    "reposerver.log.level": "debug"
  }
}'

# Redémarrer les composants
kubectl rollout restart deployment -n argocd

# Analyser les manifestes générés
argocd app manifests myapp

# Différences entre Git et cluster
argocd app diff myapp

# Historique des synchronisations
argocd app history myapp
```

## Bonnes pratiques GitOps avec ArgoCD

### 1. Structure de repository

```
gitops/
├── bootstrap/              # Applications ArgoCD de base
│   ├── argocd/            # Configuration ArgoCD elle-même
│   └── root-app.yaml      # App of Apps racine
├── platform/              # Services de plateforme
│   ├── monitoring/
│   ├── logging/
│   ├── networking/
│   └── security/
├── applications/          # Applications métier
│   ├── base/             # Configurations de base
│   └── overlays/         # Personnalisations par env
│       ├── dev/
│       ├── staging/
│       └── production/
├── clusters/             # Config spécifique par cluster
│   ├── cluster-1/
│   └── cluster-2/
└── scripts/              # Scripts utilitaires
    ├── validate.sh
    └── generate-secrets.sh
```

### 2. Gestion des environnements

```yaml
# Utiliser Kustomize pour les variations
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml

# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

patchesStrategicMerge:
  - increase-replicas.yaml
  - production-resources.yaml

configMapGenerator:
  - name: app-config
    envs:
      - config.env
```

### 3. Validation et tests

```yaml
# .gitlab-ci.yml pour validation GitOps
validate:
  stage: test
  script:
    # Validation YAML
    - find . -name "*.yaml" -o -name "*.yml" | xargs yamllint

    # Validation Kubernetes
    - find . -name "*.yaml" | xargs kubectl apply --dry-run=client -f

    # Validation ArgoCD
    - argocd app create test-app --upsert \
        --repo $CI_REPOSITORY_URL \
        --path applications/myapp \
        --dest-server https://kubernetes.default.svc \
        --dest-namespace test \
        --dry-run

    # Tests de politique (OPA)
    - opa test policies/
```

### 4. Sécurité GitOps

```yaml
# Politique OPA pour ArgoCD
package argocd

# Interdire les images sans tag spécifique
deny[msg] {
  input.kind == "Application"
  image := input.spec.source.helm.values["image.tag"]
  image == "latest"
  msg := "Les tags 'latest' sont interdits en production"
}

# Forcer les resource limits
deny[msg] {
  input.kind == "Application"
  destination := input.spec.destination.namespace
  startswith(destination, "prod")
  not input.spec.source.helm.values["resources.limits"]
  msg := "Les limits de ressources sont obligatoires en production"
}
```

### 5. Monitoring et alerting

```yaml
# PrometheusRule pour ArgoCD
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: argocd
spec:
  groups:
  - name: argocd
    interval: 30s
    rules:
    - alert: ArgoAppOutOfSync
      expr: argocd_app_info{sync_status!="Synced"} > 0
      for: 15m
      annotations:
        summary: "Application {{ $labels.name }} non synchronisée"

    - alert: ArgoAppHealthDegraded
      expr: argocd_app_info{health_status="Degraded"} > 0
      for: 15m
      annotations:
        summary: "Application {{ $labels.name }} dégradée"

    - alert: ArgoSyncFailed
      expr: rate(argocd_app_sync_total{phase="Failed"}[5m]) > 0
      for: 10m
      annotations:
        summary: "Échecs de synchronisation pour {{ $labels.name }}"
```

## Migration vers GitOps

### Plan de migration progressif

```yaml
# Phase 1: Applications stateless
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: migration-phase1
spec:
  description: Migration des apps stateless
  sourceRepos:
  - 'https://github.com/monlab/*'
  destinations:
  - namespace: 'app-*'
    server: https://kubernetes.default.svc
  roles:
  - name: developers
    policies:
    - p, proj:migration-phase1:developers, applications, get, migration-phase1/*, allow
    - p, proj:migration-phase1:developers, applications, sync, migration-phase1/*, allow

---
# Phase 2: Ajout des bases de données
# Phase 3: Services critiques
# Phase 4: Décommissionnement ancien système
```

### Script de migration

```bash
#!/bin/bash
# migrate-to-gitops.sh

# Exporter les ressources existantes
for namespace in $(kubectl get ns -o name | cut -d/ -f2); do
  echo "Exporting namespace: $namespace"
  mkdir -p exports/$namespace

  for resource in deployment service ingress configmap secret; do
    kubectl get $resource -n $namespace -o yaml > exports/$namespace/$resource.yaml
  done
done

# Nettoyer les métadonnées
find exports -name "*.yaml" -exec yq eval 'del(.metadata.resourceVersion, .metadata.uid, .metadata.selfLink)' -i {} \;

# Créer la structure GitOps
for namespace in exports/*; do
  ns=$(basename $namespace)
  mkdir -p gitops/applications/$ns/base
  cp $namespace/*.yaml gitops/applications/$ns/base/

  # Créer kustomization.yaml
  cat > gitops/applications/$ns/base/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
$(ls $namespace/*.yaml | xargs -n1 basename | sed 's/^/  - /')
EOF
done

echo "Migration complète. Vérifiez gitops/ et commitez dans Git"
```

## Conclusion

ArgoCD transforme radicalement la gestion de votre infrastructure Kubernetes en appliquant les principes GitOps. Avec votre lab MicroK8s, vous disposez maintenant d'une plateforme où :

- **Git devient la source de vérité** : Toute modification passe par Git, offrant traçabilité et auditabilité complètes
- **L'automatisation est totale** : De Git au cluster, sans intervention manuelle
- **La réconciliation est continue** : Toute dérive est automatiquement corrigée
- **Le rollback est instantané** : Retour à n'importe quel état précédent en un clic
- **La sécurité est renforcée** : RBAC, SSO, et politiques automatisées

Les compétences GitOps acquises avec ArgoCD sont hautement valorisées sur le marché, représentant l'état de l'art en matière de gestion d'infrastructure cloud-native. Votre lab devient un environnement d'apprentissage et d'expérimentation pour des pratiques utilisées dans les plus grandes organisations.

⏭️
