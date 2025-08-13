🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 Outils complémentaires

## Introduction aux outils de l'écosystème Kubernetes

Au-delà de MicroK8s et kubectl, un écosystème riche d'outils s'est développé autour de Kubernetes. Ces outils améliorent votre productivité, simplifient les tâches complexes et offrent de meilleures visualisations de votre cluster. Cette section présente les outils essentiels, organisés par catégorie d'usage.

## Interfaces graphiques et visualisation

### K9s - Terminal UI pour Kubernetes
**URL** : https://k9scli.io/
**Installation** : `brew install k9s` ou téléchargement direct

#### Caractéristiques principales
- **Navigation intuitive** : Vim-like keybindings
- **Vue temps réel** : Rafraîchissement automatique
- **Actions rapides** : Logs, exec, edit, delete
- **Resource monitoring** : CPU, mémoire en direct
- **Multi-cluster** : Switch facile entre contextes

#### Commandes essentielles K9s
```
# Lancement
k9s

# Navigation de base
:pods       # Liste des pods
:svc        # Liste des services
:deploy     # Liste des deployments
:ns         # Change namespace
ctrl+a      # Affiche toutes les ressources
/           # Recherche
?           # Aide
```

#### Cas d'usage idéaux
- Surveillance quotidienne du cluster
- Debugging rapide
- Navigation exploratoire
- Alternative à plusieurs commandes kubectl

### Lens - The Kubernetes IDE
**URL** : https://k8slens.dev/
**Type** : Application desktop (Windows, Mac, Linux)

#### Fonctionnalités clés
- **Interface graphique complète** : Point-and-click
- **Multi-cluster management** : Tous vos clusters en un endroit
- **Metrics intégrées** : Sans configuration Prometheus
- **Terminal intégré** : kubectl direct
- **Extensions** : Marketplace d'extensions

#### Organisation dans Lens
1. **Catalog** : Liste de tous vos clusters
2. **Workloads** : Pods, Deployments, Jobs
3. **Network** : Services, Ingresses, Endpoints
4. **Storage** : PV, PVC, StorageClasses
5. **Configuration** : ConfigMaps, Secrets

#### Points d'attention
- Version gratuite vs payante
- Consommation de ressources sur le poste
- Synchronisation avec kubeconfig

### Octant - Web UI pour Kubernetes
**URL** : https://octant.dev/
**Installation** : `brew install octant`

#### Avantages d'Octant
- **Open source** : Totalement gratuit
- **Mode local** : Pas d'installation sur le cluster
- **Visualisations** : Graphiques de dépendances
- **Extensible** : Plugins en Go ou JavaScript
- **Port-forwarding UI** : Interface pour les forwards

#### Utilisation basique
```bash
# Lancement pour le cluster actuel
octant

# Avec un kubeconfig spécifique
octant --kubeconfig ~/.kube/config

# Sur un port spécifique
octant --listener-addr 0.0.0.0:8900
```

### Kubernetes Dashboard (officiel)
**URL** : https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

#### Activation dans MicroK8s
```bash
# Installation simple
microk8s enable dashboard

# Accès au dashboard
microk8s dashboard-proxy
```

#### Fonctionnalités natives
- Vue d'ensemble du cluster
- Gestion des ressources
- Logs des conteneurs
- Métriques basiques
- Exec dans les pods

## Outils de ligne de commande améliorés

### kubectl plugins via Krew
**URL** : https://krew.sigs.k8s.io/
**Installation** : Script d'installation sur le site

#### Plugins essentiels pour MicroK8s

##### kubectl-ns et kubectl-ctx
```bash
# Installation
kubectl krew install ns
kubectl krew install ctx

# Utilisation
kubectl ns          # Liste et change de namespace
kubectl ctx         # Liste et change de contexte
```

##### kubectl-tree
```bash
# Installation
kubectl krew install tree

# Affiche l'arbre des ressources
kubectl tree deployment my-app
```

##### kubectl-debug
```bash
# Installation
kubectl krew install debug

# Debug un pod avec un conteneur ephémère
kubectl debug my-pod -it --image=busybox
```

### kubectx et kubens
**URL** : https://github.com/ahmetb/kubectx
**Installation** : `brew install kubectx`

#### Utilisation quotidienne
```bash
# Changer de contexte
kubectx microk8s
kubectx -           # Retour au contexte précédent

# Changer de namespace
kubens kube-system
kubens -            # Retour au namespace précédent
```

### stern - Multi-pod log tailing
**URL** : https://github.com/stern/stern
**Installation** : `brew install stern`

#### Cas d'usage puissants
```bash
# Logs de tous les pods d'un deployment
stern deployment/my-app

# Logs avec filtre sur le nom
stern "web-.*"

# Logs multi-conteneurs
stern my-pod --all-namespaces

# Avec timestamps et couleurs
stern my-app --timestamps --since 1h
```

### k3sup - Installation facile de K3s/MicroK8s
**URL** : https://k3sup.dev/
**Par** : Alex Ellis

#### Utilisation pour MicroK8s
```bash
# Installation locale
k3sup install --local --k3s-channel stable

# Installation sur serveur distant
k3sup install --ip 192.168.1.100 --user ubuntu

# Ajout d'un node
k3sup join --ip 192.168.1.101 --server-ip 192.168.1.100
```

## Gestion des manifestes et configurations

### Helm - Package Manager pour Kubernetes
**URL** : https://helm.sh/
**Installation** : `brew install helm`

#### Concepts de base Helm
- **Charts** : Packages d'applications
- **Repositories** : Stockage de charts
- **Values** : Configuration des charts
- **Releases** : Instances déployées

#### Commandes essentielles
```bash
# Ajout d'un repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Recherche de charts
helm search repo wordpress

# Installation d'un chart
helm install my-wordpress bitnami/wordpress

# Mise à jour
helm upgrade my-wordpress bitnami/wordpress

# Désinstallation
helm uninstall my-wordpress
```

#### Helm avec MicroK8s
```bash
# Activation de Helm3 dans MicroK8s
microk8s enable helm3

# Utilisation
microk8s helm3 repo add stable https://charts.helm.sh/stable
microk8s helm3 install my-release stable/mysql
```

### Kustomize - Configuration Management
**URL** : https://kustomize.io/
**Intégré dans kubectl** : Depuis v1.14

#### Structure Kustomize
```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml

overlays/
├── development/
│   └── kustomization.yaml
├── staging/
│   └── kustomization.yaml
└── production/
    └── kustomization.yaml
```

#### Utilisation de base
```bash
# Application directe
kubectl apply -k overlays/development

# Génération du YAML
kubectl kustomize overlays/production

# Avec MicroK8s
microk8s kubectl apply -k .
```

### Jsonnet - Data Templating Language
**URL** : https://jsonnet.org/
**Installation** : `brew install jsonnet`

#### Exemple simple Jsonnet
```jsonnet
// deployment.jsonnet
local deployment = {
  apiVersion: "apps/v1",
  kind: "Deployment",
  metadata: {
    name: "my-app",
  },
  spec: {
    replicas: 3,
    // ... reste de la config
  }
};

deployment
```

#### Compilation vers YAML
```bash
jsonnet deployment.jsonnet | kubectl apply -f -
```

## Outils de développement et CI/CD

### Skaffold - Development Workflow
**URL** : https://skaffold.dev/
**Installation** : `brew install skaffold`

#### Configuration Skaffold
```yaml
# skaffold.yaml
apiVersion: skaffold/v2beta28
kind: Config
build:
  artifacts:
  - image: my-app
deploy:
  kubectl:
    manifests:
    - k8s/*.yaml
```

#### Workflow de développement
```bash
# Développement avec hot-reload
skaffold dev

# Build et deploy une fois
skaffold run

# Debug avec breakpoints
skaffold debug
```

### Tilt - Development Environment
**URL** : https://tilt.dev/
**Installation** : `brew install tilt`

#### Tiltfile exemple
```python
# Tiltfile
docker_build('my-app', '.')
k8s_yaml('deployment.yaml')
k8s_resource('my-app', port_forwards='8080:8080')
```

#### Lancement
```bash
tilt up
# Interface web sur http://localhost:10350
```

### Garden - Development Platform
**URL** : https://garden.io/
**Installation** : `brew install garden-cli`

#### Configuration Garden
```yaml
# garden.yml
kind: Project
name: my-project
providers:
  - name: kubernetes
    context: microk8s
```

## Sécurité et conformité

### Falco - Runtime Security
**URL** : https://falco.org/
**Type** : CNCF Project

#### Installation sur MicroK8s
```bash
# Via Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco
```

#### Ce que Falco détecte
- Shells dans les conteneurs
- Modifications de fichiers sensibles
- Connexions réseau suspectes
- Escalade de privilèges

### Trivy - Vulnerability Scanner
**URL** : https://trivy.dev/
**Installation** : `brew install trivy`

#### Scans avec Trivy
```bash
# Scan d'une image
trivy image nginx:latest

# Scan de fichiers K8s
trivy config k8s-manifests/

# Scan du cluster
trivy k8s --report summary cluster
```

### OPA - Open Policy Agent
**URL** : https://www.openpolicyagent.org/
**Type** : CNCF Project

#### Cas d'usage OPA
- Admission control
- Policy as Code
- Compliance automatisée
- Gestion des ressources

### Kubescape - Security Posture Scanner
**URL** : https://kubescape.io/
**Installation** : `brew install kubescape`

#### Utilisation basique
```bash
# Scan complet du cluster
kubescape scan

# Scan avec framework NSA
kubescape scan framework nsa

# Scan de manifestes locaux
kubescape scan *.yaml
```

## Monitoring et observabilité

### Prometheus Operator
**URL** : https://prometheus-operator.dev/

#### Installation simplifiée
```bash
# Via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack
```

#### Composants inclus
- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- Kube State Metrics

### Pixie - Instant Kubernetes Observability
**URL** : https://px.dev/
**Acquisition** : New Relic

#### Caractéristiques uniques
- Zero instrumentation
- eBPF-based
- Données en temps réel
- Language agnostic

### Kube-bench - CIS Benchmark
**URL** : https://github.com/aquasecurity/kube-bench

#### Exécution d'un audit
```bash
# Installation
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Consultation des résultats
kubectl logs job/kube-bench
```

## Gestion du stockage

### Velero - Backup and Restore
**URL** : https://velero.io/
**Installation** : Via Helm ou CLI

#### Configuration de base
```bash
# Installation CLI
brew install velero

# Installation sur le cluster
velero install --provider aws --bucket my-bucket

# Création d'un backup
velero backup create my-backup

# Restauration
velero restore create --from-backup my-backup
```

### Rook - Storage Orchestrator
**URL** : https://rook.io/
**Type** : CNCF Project

#### Déploiement Rook/Ceph
```bash
# Operator
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml

# Cluster Ceph
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/cluster.yaml
```

### Longhorn - Distributed Storage
**URL** : https://longhorn.io/
**Par** : Rancher/SUSE

#### Avantages pour MicroK8s
- Léger
- Interface web incluse
- Snapshots et backups
- Réplication simple

## Networking et service mesh

### Istio - Service Mesh
**URL** : https://istio.io/

#### Installation avec MicroK8s
```bash
# Addon Istio
microk8s enable istio

# Vérification
microk8s kubectl get pods -n istio-system
```

### Linkerd - Lightweight Service Mesh
**URL** : https://linkerd.io/
**Caractéristique** : Plus léger qu'Istio

#### Installation rapide
```bash
# CLI
brew install linkerd

# Installation sur cluster
linkerd install | kubectl apply -f -
linkerd check
```

### Cilium - eBPF-based Networking
**URL** : https://cilium.io/
**Performance** : Très haute avec eBPF

#### Avec MicroK8s
```bash
# Remplace le CNI par défaut
microk8s enable cilium
```

## Testing et validation

### Kubectl-validate
**URL** : https://github.com/kubernetes/kubectl-validate

#### Validation de manifestes
```bash
# Validation simple
kubectl validate my-deployment.yaml

# Avec version d'API spécifique
kubectl validate --version 1.27 my-manifest.yaml
```

### Kubeconform - Manifest Validation
**URL** : https://github.com/yannh/kubeconform
**Installation** : `brew install kubeconform`

#### Utilisation
```bash
# Validation simple
kubeconform deployment.yaml

# Validation stricte
kubeconform -strict -verbose manifests/
```

### Chaos Mesh - Chaos Engineering
**URL** : https://chaos-mesh.org/
**Type** : CNCF Project

#### Types de chaos
- Pod failures
- Network delays
- CPU/Memory stress
- I/O delays
- Time skew

### Litmus - Chaos Engineering Platform
**URL** : https://litmuschaos.io/
**Installation** : Via Helm

#### Expériences disponibles
- Pod delete
- Container kill
- Node drain
- Disk fill
- Network corruption

## Utilitaires divers

### kubectl-neat - Clean YAML Output
**Installation** : `kubectl krew install neat`

#### Utilisation
```bash
# Output nettoyé
kubectl get pod my-pod -o yaml | kubectl neat

# Supprime les champs générés
kubectl get deployment -o yaml | kubectl neat
```

### kubecolor - Colorized kubectl
**URL** : https://github.com/hidetatz/kubecolor
**Installation** : `brew install kubecolor`

#### Configuration
```bash
# Alias recommandé
alias kubectl="kubecolor"

# Utilisation normale
kubecolor get pods
kubecolor describe service my-svc
```

### kube-ps1 - Kubernetes Prompt
**URL** : https://github.com/jonmosco/kube-ps1

#### Configuration bash/zsh
```bash
# Dans .bashrc ou .zshrc
source "/opt/kube-ps1/kube-ps1.sh"
PS1='$(kube_ps1)'$PS1
```

### kubefwd - Port Forwarding
**URL** : https://github.com/txn2/kubefwd
**Installation** : `brew install kubefwd`

#### Forwarding de services
```bash
# Forward tous les services d'un namespace
sudo kubefwd svc -n my-namespace

# Services accessibles par nom
curl http://my-service:8080
```

## Choix d'outils selon les besoins

### Pour débuter
1. **K9s** : Navigation visuelle du cluster
2. **Helm** : Installation d'applications
3. **kubectl plugins** : ns, ctx pour la navigation
4. **Lens** : Si vous préférez une GUI

### Pour le développement
1. **Skaffold ou Tilt** : Hot-reload et développement
2. **Stern** : Logs multi-pods
3. **kubefwd** : Accès aux services

### Pour la production
1. **Velero** : Backups
2. **Trivy** : Sécurité
3. **Prometheus Operator** : Monitoring complet
4. **Falco** : Runtime security

### Pour l'apprentissage
1. **K9s** : Exploration interactive
2. **kubectl-tree** : Comprendre les relations
3. **Octant** : Visualisations pédagogiques
4. **kubecolor** : Output plus lisible

## Installation et gestion des outils

### Gestionnaires de packages recommandés

#### Homebrew (Mac/Linux)
```bash
# Installation de la plupart des outils
brew install k9s helm kubectl stern
```

#### Chocolatey (Windows)
```powershell
# Installation sur Windows
choco install k9s kubernetes-helm
```

#### Package managers spécifiques
- **Krew** : Pour les plugins kubectl
- **Helm** : Pour les charts Kubernetes
- **arkade** : Installation d'outils cloud-native

### Arkade - Marketplace for DevOps tools
**URL** : https://arkade.dev/
**Installation** : `curl -sLS https://get.arkade.dev | sudo sh`

#### Utilisation d'arkade
```bash
# Liste des outils disponibles
arkade get --help

# Installation d'outils
arkade get kubectl
arkade get k9s
arkade get helm
arkade get k3sup

# Installation de charts
arkade install prometheus
arkade install grafana
```

## Maintenance et mise à jour

### Stratégie de mise à jour
1. **Outils critiques** : Mise à jour régulière (kubectl, helm)
2. **Outils de dev** : Selon les besoins
3. **Outils de sécurité** : Toujours à jour
4. **Outils de confort** : Quand vous avez le temps

### Vérification des versions
```bash
# Script de vérification
for tool in kubectl helm k9s stern; do
  echo "$tool: $($tool version --short 2>/dev/null || $tool --version)"
done
```

---

Ces outils complémentaires transforment votre expérience avec MicroK8s. Commencez par quelques outils essentiels, puis élargissez votre boîte à outils progressivement selon vos besoins. Chaque outil a sa place dans votre workflow, mais évitez la surcharge : mieux vaut maîtriser quelques outils que d'en avoir beaucoup mal configurés.

⏭️
