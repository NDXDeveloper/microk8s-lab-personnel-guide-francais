🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A - Scripts d'installation automatisée

## Introduction aux Scripts d'Installation

L'installation manuelle de MicroK8s et de ses composants peut être répétitive et source d'erreurs, surtout lorsqu'on doit recréer régulièrement des environnements de lab. Cette annexe fournit des scripts d'installation automatisée qui garantissent une configuration cohérente et complète de votre cluster MicroK8s.

### Pourquoi Automatiser l'Installation ?

L'automatisation de l'installation présente plusieurs avantages essentiels :

- **Reproductibilité** : Obtenir exactement la même configuration à chaque installation
- **Gain de temps** : Réduire une installation de 30-45 minutes à moins de 10 minutes
- **Réduction des erreurs** : Éliminer les oublis et les fautes de frappe
- **Documentation vivante** : Le script lui-même documente votre configuration
- **Idempotence** : Pouvoir relancer le script sans craindre de casser l'existant

## Script Principal d'Installation

### Script All-in-One pour Ubuntu/Debian

Ce script complet installe MicroK8s avec les addons essentiels pour un lab fonctionnel.

```bash
#!/bin/bash
#
# Script d'installation automatisée MicroK8s pour Lab Personnel
# Compatible : Ubuntu 20.04+, Debian 11+
# Version : 1.0
# Dernière mise à jour : 2025
#

set -euo pipefail  # Arrêt immédiat en cas d'erreur

# ===== CONFIGURATION =====
# Modifiez ces variables selon vos besoins

MICROK8S_CHANNEL="1.29/stable"  # Version de MicroK8s à installer
USER_NAME="${SUDO_USER:-$USER}"  # Utilisateur qui utilisera MicroK8s
ENABLE_DASHBOARD="true"          # Activer le dashboard Kubernetes
ENABLE_INGRESS="true"           # Activer l'Ingress Controller
ENABLE_DNS="true"                # Activer CoreDNS
ENABLE_STORAGE="true"            # Activer le stockage local
ENABLE_REGISTRY="true"           # Activer le registry privé
ENABLE_METRICS="true"            # Activer metrics-server
DOMAIN_NAME=""                   # Votre domaine (optionnel)

# Couleurs pour l'affichage
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# ===== FONCTIONS UTILITAIRES =====

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "Ce script doit être exécuté avec sudo"
        exit 1
    fi
}

check_os() {
    if [[ ! -f /etc/os-release ]]; then
        log_error "Impossible de déterminer l'OS"
        exit 1
    fi

    . /etc/os-release

    if [[ "$ID" != "ubuntu" && "$ID" != "debian" ]]; then
        log_error "OS non supporté. Ubuntu ou Debian requis."
        exit 1
    fi

    log_info "OS détecté : $PRETTY_NAME"
}

# ===== INSTALLATION DE BASE =====

install_prerequisites() {
    log_info "Installation des prérequis système..."

    apt-get update
    apt-get install -y \
        curl \
        wget \
        snapd \
        jq \
        net-tools \
        ca-certificates \
        apt-transport-https

    # Assurer que snapd est démarré
    systemctl enable --now snapd.socket
    systemctl enable --now snapd.service

    log_info "Prérequis installés avec succès"
}

install_microk8s() {
    log_info "Installation de MicroK8s (channel: $MICROK8S_CHANNEL)..."

    # Installation via snap
    snap install microk8s --classic --channel=$MICROK8S_CHANNEL

    # Attendre que MicroK8s soit prêt
    log_info "Attente du démarrage de MicroK8s..."
    microk8s status --wait-ready

    log_info "MicroK8s installé avec succès"
}

configure_user_permissions() {
    log_info "Configuration des permissions pour l'utilisateur $USER_NAME..."

    # Ajouter l'utilisateur au groupe microk8s
    usermod -a -G microk8s $USER_NAME

    # Créer le répertoire .kube si nécessaire
    mkdir -p /home/$USER_NAME/.kube

    # Donner les permissions appropriées
    chown -f -R $USER_NAME:$USER_NAME /home/$USER_NAME/.kube

    log_info "Permissions configurées. L'utilisateur devra se reconnecter pour appliquer les changements."
}

# ===== CONFIGURATION DES ADDONS =====

enable_addons() {
    log_info "Activation des addons MicroK8s..."

    # DNS (CoreDNS) - Toujours nécessaire
    if [[ "$ENABLE_DNS" == "true" ]]; then
        log_info "Activation de CoreDNS..."
        microk8s enable dns
    fi

    # Dashboard
    if [[ "$ENABLE_DASHBOARD" == "true" ]]; then
        log_info "Activation du Dashboard Kubernetes..."
        microk8s enable dashboard
    fi

    # Ingress Controller
    if [[ "$ENABLE_INGRESS" == "true" ]]; then
        log_info "Activation de l'Ingress Controller NGINX..."
        microk8s enable ingress
    fi

    # Storage
    if [[ "$ENABLE_STORAGE" == "true" ]]; then
        log_info "Activation du stockage persistant local..."
        microk8s enable hostpath-storage
    fi

    # Registry privé
    if [[ "$ENABLE_REGISTRY" == "true" ]]; then
        log_info "Activation du registry Docker privé..."
        microk8s enable registry
    fi

    # Metrics Server
    if [[ "$ENABLE_METRICS" == "true" ]]; then
        log_info "Activation du metrics-server..."
        microk8s enable metrics-server
    fi

    log_info "Addons activés avec succès"
}

# ===== CONFIGURATION KUBECTL =====

setup_kubectl() {
    log_info "Configuration de kubectl..."

    # Installer kubectl si non présent
    if ! command -v kubectl &> /dev/null; then
        log_info "Installation de kubectl..."
        snap install kubectl --classic
    fi

    # Créer l'alias microk8s kubectl
    cat >> /home/$USER_NAME/.bashrc << 'EOF'

# Alias MicroK8s
alias kubectl='microk8s kubectl'
alias k='microk8s kubectl'
alias helm='microk8s helm'

# Autocomplétion kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k
EOF

    # Exporter la configuration kubeconfig
    microk8s config > /home/$USER_NAME/.kube/config
    chown $USER_NAME:$USER_NAME /home/$USER_NAME/.kube/config
    chmod 600 /home/$USER_NAME/.kube/config

    log_info "kubectl configuré avec succès"
}

# ===== VÉRIFICATIONS POST-INSTALLATION =====

verify_installation() {
    log_info "Vérification de l'installation..."

    # Vérifier le statut du cluster
    if microk8s status | grep -q "microk8s is running"; then
        log_info "✓ MicroK8s est en cours d'exécution"
    else
        log_error "✗ MicroK8s n'est pas en cours d'exécution"
        return 1
    fi

    # Vérifier les nodes
    log_info "Nodes du cluster :"
    microk8s kubectl get nodes

    # Vérifier les pods système
    log_info "Pods système :"
    microk8s kubectl get pods --all-namespaces

    # Afficher les addons activés
    log_info "Addons activés :"
    microk8s status | grep enabled

    return 0
}

# ===== INFORMATIONS POST-INSTALLATION =====

display_access_info() {
    echo ""
    echo "========================================="
    echo "   Installation MicroK8s Terminée !"
    echo "========================================="
    echo ""
    log_info "Prochaines étapes :"
    echo ""
    echo "1. Reconnectez-vous ou rechargez votre session :"
    echo "   $ newgrp microk8s"
    echo ""
    echo "2. Vérifiez l'installation :"
    echo "   $ kubectl get nodes"
    echo "   $ kubectl get pods --all-namespaces"
    echo ""

    if [[ "$ENABLE_DASHBOARD" == "true" ]]; then
        echo "3. Accès au Dashboard :"
        echo "   $ kubectl port-forward -n kube-system service/kubernetes-dashboard 8443:443"
        echo "   Puis ouvrez : https://localhost:8443"
        echo ""
        echo "   Token d'accès :"
        echo "   $ kubectl describe secret -n kube-system microk8s-dashboard-token"
        echo ""
    fi

    if [[ "$ENABLE_REGISTRY" == "true" ]]; then
        echo "4. Registry privé disponible :"
        echo "   localhost:32000"
        echo ""
    fi

    echo "========================================="
}

# ===== FONCTION PRINCIPALE =====

main() {
    log_info "Démarrage de l'installation automatisée de MicroK8s..."

    # Vérifications préliminaires
    check_root
    check_os

    # Installation
    install_prerequisites
    install_microk8s
    configure_user_permissions

    # Configuration
    enable_addons
    setup_kubectl

    # Vérification
    if verify_installation; then
        display_access_info
        log_info "Installation terminée avec succès !"
    else
        log_error "L'installation a rencontré des problèmes. Vérifiez les logs."
        exit 1
    fi
}

# ===== POINT D'ENTRÉE =====

# Gestion du trap pour nettoyer en cas d'erreur
trap 'log_error "Une erreur est survenue. Arrêt du script."' ERR

# Lancer l'installation
main "$@"
```

## Scripts Complémentaires

### Script de Configuration Post-Installation

Ce script configure les éléments avancés après l'installation de base.

```bash
#!/bin/bash
#
# Script de configuration post-installation MicroK8s
# À exécuter après le script d'installation principal
#

set -euo pipefail

# Configuration
CERT_MANAGER_VERSION="v1.13.0"
METALLB_RANGE="192.168.1.200-192.168.1.210"  # Adapter à votre réseau

# Couleurs
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

# Installation de Cert-Manager pour les certificats SSL
install_cert_manager() {
    log_info "Installation de Cert-Manager..."

    # Appliquer les CRDs
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.crds.yaml

    # Créer le namespace
    kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl apply -f -

    # Installer Cert-Manager
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml

    # Attendre que Cert-Manager soit prêt
    log_info "Attente du démarrage de Cert-Manager..."
    kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s

    log_info "Cert-Manager installé avec succès"
}

# Configuration de MetalLB pour le LoadBalancer
setup_metallb() {
    log_info "Configuration de MetalLB..."

    # Activer MetalLB
    microk8s enable metallb:$METALLB_RANGE

    log_info "MetalLB configuré avec la plage IP : $METALLB_RANGE"
}

# Configuration de l'observabilité
setup_observability() {
    log_info "Configuration de l'observabilité..."

    # Activer l'addon observability (Prometheus, Grafana, Loki)
    microk8s enable observability

    log_info "Stack d'observabilité activée (Prometheus, Grafana, Loki)"
    log_info "Grafana sera accessible sur le port 3000"
}

# Création d'un namespace pour les applications
create_app_namespace() {
    log_info "Création du namespace 'apps' pour vos applications..."

    kubectl create namespace apps --dry-run=client -o yaml | kubectl apply -f -

    # Définir comme namespace par défaut
    kubectl config set-context --current --namespace=apps

    log_info "Namespace 'apps' créé et défini par défaut"
}

# Configuration des limites de ressources par défaut
setup_resource_limits() {
    log_info "Configuration des limites de ressources par défaut..."

    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: apps
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 100m
    type: Container
EOF

    log_info "Limites de ressources configurées"
}

# Fonction principale
main() {
    log_info "Démarrage de la configuration post-installation..."

    # Vérifier que MicroK8s est en cours d'exécution
    if ! microk8s status | grep -q "microk8s is running"; then
        log_warn "MicroK8s n'est pas en cours d'exécution. Démarrage..."
        microk8s start
        microk8s status --wait-ready
    fi

    # Configurations
    install_cert_manager
    setup_metallb
    setup_observability
    create_app_namespace
    setup_resource_limits

    log_info "Configuration post-installation terminée !"

    echo ""
    echo "===== Ressources Configurées ====="
    echo "• Cert-Manager : Gestion automatique des certificats SSL"
    echo "• MetalLB : LoadBalancer avec la plage $METALLB_RANGE"
    echo "• Observabilité : Prometheus, Grafana (port 3000), Loki"
    echo "• Namespace 'apps' : Pour vos applications"
    echo "• Limites de ressources : Configurées par défaut"
    echo "=================================="
}

main "$@"
```

### Script de Vérification Santé

Script pour vérifier régulièrement l'état de votre cluster.

```bash
#!/bin/bash
#
# Script de vérification santé MicroK8s
# À exécuter régulièrement pour monitorer l'état du cluster
#

set -euo pipefail

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Compteurs
TESTS_TOTAL=0
TESTS_PASSED=0
TESTS_FAILED=0

# Fonctions d'affichage
print_header() {
    echo ""
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    echo -e "${BLUE}  $1${NC}"
    echo -e "${BLUE}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
}

test_pass() {
    echo -e "${GREEN}✓${NC} $1"
    ((TESTS_PASSED++))
    ((TESTS_TOTAL++))
}

test_fail() {
    echo -e "${RED}✗${NC} $1"
    ((TESTS_FAILED++))
    ((TESTS_TOTAL++))
}

test_warn() {
    echo -e "${YELLOW}⚠${NC} $1"
}

# Tests système
check_system() {
    print_header "Vérification Système"

    # Mémoire disponible
    MEM_AVAILABLE=$(free -m | awk 'NR==2{printf "%.1f", $7/1024}')
    if (( $(echo "$MEM_AVAILABLE > 1" | bc -l) )); then
        test_pass "Mémoire disponible : ${MEM_AVAILABLE}GB"
    else
        test_fail "Mémoire insuffisante : ${MEM_AVAILABLE}GB (minimum 1GB recommandé)"
    fi

    # Espace disque
    DISK_USAGE=$(df / | awk 'NR==2{print $5}' | sed 's/%//')
    if [[ $DISK_USAGE -lt 80 ]]; then
        test_pass "Espace disque utilisé : ${DISK_USAGE}%"
    else
        test_warn "Espace disque élevé : ${DISK_USAGE}%"
    fi

    # CPU Load
    LOAD_AVG=$(cat /proc/loadavg | awk '{print $1}')
    CPU_COUNT=$(nproc)
    if (( $(echo "$LOAD_AVG < $CPU_COUNT" | bc -l) )); then
        test_pass "Charge CPU : $LOAD_AVG (${CPU_COUNT} cores)"
    else
        test_warn "Charge CPU élevée : $LOAD_AVG (${CPU_COUNT} cores)"
    fi
}

# Tests MicroK8s
check_microk8s() {
    print_header "Vérification MicroK8s"

    # Statut MicroK8s
    if microk8s status | grep -q "microk8s is running"; then
        test_pass "MicroK8s est en cours d'exécution"
    else
        test_fail "MicroK8s n'est pas en cours d'exécution"
        return
    fi

    # Node status
    NODE_STATUS=$(kubectl get nodes -o json | jq -r '.items[0].status.conditions[] | select(.type=="Ready") | .status')
    if [[ "$NODE_STATUS" == "True" ]]; then
        test_pass "Node est prêt"
    else
        test_fail "Node n'est pas prêt"
    fi

    # Pods système
    FAILED_PODS=$(kubectl get pods --all-namespaces -o json | jq '[.items[] | select(.status.phase != "Running" and .status.phase != "Succeeded")] | length')
    if [[ $FAILED_PODS -eq 0 ]]; then
        test_pass "Tous les pods système sont en cours d'exécution"
    else
        test_fail "$FAILED_PODS pods système en erreur"
        kubectl get pods --all-namespaces | grep -v Running | grep -v Succeeded | grep -v STATUS
    fi
}

# Tests des addons
check_addons() {
    print_header "Vérification des Addons"

    # Liste des addons essentiels
    ESSENTIAL_ADDONS=("dns" "hostpath-storage")

    for addon in "${ESSENTIAL_ADDONS[@]}"; do
        if microk8s status | grep -q "$addon: enabled"; then
            test_pass "Addon '$addon' activé"
        else
            test_fail "Addon '$addon' non activé"
        fi
    done

    # Vérifier les addons optionnels
    OPTIONAL_ADDONS=("dashboard" "ingress" "metrics-server" "registry")

    for addon in "${OPTIONAL_ADDONS[@]}"; do
        if microk8s status | grep -q "$addon: enabled"; then
            test_pass "Addon '$addon' activé"
        else
            test_warn "Addon '$addon' non activé (optionnel)"
        fi
    done
}

# Tests réseau
check_network() {
    print_header "Vérification Réseau"

    # DNS cluster
    if kubectl run -i --rm --restart=Never test-dns --image=busybox:1.28 -- nslookup kubernetes.default 2>/dev/null | grep -q "Address"; then
        test_pass "DNS cluster fonctionnel"
    else
        test_fail "DNS cluster non fonctionnel"
    fi

    # Connectivité externe
    if kubectl run -i --rm --restart=Never test-external --image=busybox:1.28 -- wget -q -O- http://www.google.com 2>/dev/null | grep -q "google"; then
        test_pass "Connectivité externe OK"
    else
        test_warn "Pas de connectivité externe"
    fi
}

# Tests de stockage
check_storage() {
    print_header "Vérification Stockage"

    # StorageClass par défaut
    DEFAULT_SC=$(kubectl get storageclass -o json | jq -r '.items[] | select(.metadata.annotations."storageclass.kubernetes.io/is-default-class"=="true") | .metadata.name')
    if [[ -n "$DEFAULT_SC" ]]; then
        test_pass "StorageClass par défaut : $DEFAULT_SC"
    else
        test_fail "Aucune StorageClass par défaut"
    fi

    # PersistentVolumes
    PV_COUNT=$(kubectl get pv --no-headers 2>/dev/null | wc -l)
    test_warn "PersistentVolumes : $PV_COUNT"

    # PersistentVolumeClaims
    PVC_COUNT=$(kubectl get pvc --all-namespaces --no-headers 2>/dev/null | wc -l)
    test_warn "PersistentVolumeClaims : $PVC_COUNT"
}

# Rapport final
print_summary() {
    print_header "Résumé"

    echo ""
    echo "Tests exécutés : $TESTS_TOTAL"
    echo -e "${GREEN}Réussis : $TESTS_PASSED${NC}"
    echo -e "${RED}Échoués : $TESTS_FAILED${NC}"

    echo ""
    if [[ $TESTS_FAILED -eq 0 ]]; then
        echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
        echo -e "${GREEN}  ✓ Cluster en bonne santé !${NC}"
        echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    else
        echo -e "${RED}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
        echo -e "${RED}  ✗ Des problèmes ont été détectés${NC}"
        echo -e "${RED}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
    fi
}

# Fonction principale
main() {
    echo -e "${BLUE}╔══════════════════════════════════════════╗${NC}"
    echo -e "${BLUE}║     Vérification Santé MicroK8s Lab      ║${NC}"
    echo -e "${BLUE}╚══════════════════════════════════════════╝${NC}"

    check_system
    check_microk8s
    check_addons
    check_network
    check_storage

    print_summary
}

# Exécution
main "$@"
```

## Script de Désinstallation Propre

Pour nettoyer complètement votre système si nécessaire.

```bash
#!/bin/bash
#
# Script de désinstallation complète de MicroK8s
# ATTENTION : Supprime toutes les données !
#

set -euo pipefail

RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
NC='\033[0m'

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

# Confirmation
echo -e "${RED}╔══════════════════════════════════════════╗${NC}"
echo -e "${RED}║           ATTENTION - DANGER             ║${NC}"
echo -e "${RED}║                                          ║${NC}"
echo -e "${RED}║  Ce script va supprimer complètement     ║${NC}"
echo -e "${RED}║  MicroK8s et TOUTES les données !        ║${NC}"
echo -e "${RED}║                                          ║${NC}"
echo -e "${RED}║  Cette action est IRRÉVERSIBLE !         ║${NC}"
echo -e "${RED}╚══════════════════════════════════════════╝${NC}"
echo ""

read -p "Êtes-vous sûr de vouloir continuer ? (tapez 'OUI' pour confirmer) : " CONFIRM

if [[ "$CONFIRM" != "OUI" ]]; then
    log_info "Désinstallation annulée"
    exit 0
fi

# Vérification root
if [[ $EUID -ne 0 ]]; then
    log_warn "Ce script doit être exécuté avec sudo"
    exit 1
fi

log_info "Début de la désinstallation..."

# Arrêt de MicroK8s
if command -v microk8s &> /dev/null; then
    log_info "Arrêt de MicroK8s..."
    microk8s stop || true

    # Reset du cluster
    log_info "Reset du cluster..."
    microk8s reset --destroy-storage || true
fi

# Suppression via snap
log_info "Suppression de MicroK8s..."
snap remove microk8s --purge || true

# Nettoyage des répertoires
log_info "Nettoyage des répertoires..."
rm -rf /var/snap/microk8s
rm -rf /var/lib/microk8s
rm -rf /root/.kube
rm -rf /home/*/.kube

# Suppression des groupes
log_info "Suppression des groupes..."
groupdel microk8s 2>/dev/null || true

# Nettoyage des alias
log_info "Nettoyage des alias bash..."
for user_home in /home/*; do
    if [[ -f "$user_home/.bashrc" ]]; then
        sed -i '/# Alias MicroK8s/,/complete -F __start_kubectl k/d' "$user_home/.bashrc"
    fi
done

# Nettoyage iptables (optionnel, à décommenter si nécessaire)
# log_info "Nettoyage des règles iptables..."
# iptables -F
# iptables -X
# iptables -t nat -F
# iptables -t nat -X

log_info "Désinstallation terminée"
echo ""
echo -e "${GREEN}MicroK8s a été complètement supprimé du système${NC}"
```

## Guide d'Utilisation des Scripts

### Prérequis Système

Avant d'exécuter les scripts, assurez-vous que votre système répond aux exigences minimales :

- **OS** : Ubuntu 20.04+ ou Debian 11+
- **RAM** : Minimum 4 GB (8 GB recommandé)
- **CPU** : 2 cores minimum (4 cores recommandé)
- **Disque** : 20 GB d'espace libre minimum
- **Réseau** : Connexion Internet pour télécharger les packages

### Étapes d'Installation

#### 1. Téléchargement du Script

```bash
# Créer un répertoire pour les scripts
mkdir -p ~/microk8s-scripts
cd ~/microk8s-scripts

# Télécharger le script principal (créez le fichier et copiez le contenu)
nano install-microk8s.sh

# Rendre le script exécutable
chmod +x install-microk8s.sh
```

#### 2. Configuration Préalable

Avant de lancer le script, éditez les variables de configuration en début de fichier :

```bash
# Ouvrir le script pour édition
nano install-microk8s.sh

# Modifier les variables selon vos besoins :
# - MICROK8S_CHANNEL : version de MicroK8s
# - ENABLE_* : activer/désactiver les addons
# - DOMAIN_NAME : votre domaine (si applicable)
```

#### 3. Exécution

```bash
# Lancer l'installation avec sudo
sudo ./install-microk8s.sh

# Le script va :
# 1. Vérifier les prérequis
# 2. Installer MicroK8s
# 3. Configurer les permissions
# 4. Activer les addons
# 5. Configurer kubectl
# 6. Afficher les informations d'accès
```

#### 4. Post-Installation

Après l'installation principale :

```bash
# Se reconnecter au groupe microk8s
newgrp microk8s

# Vérifier l'installation
kubectl get nodes
kubectl get pods --all-namespaces

# Exécuter le script de configuration post-installation (optionnel)
sudo ./post-install-config.sh

# Lancer une vérification santé
./health-check.sh
```

### Personnalisation des Scripts

#### Adaptation au Réseau Local

Pour adapter les scripts à votre réseau local, modifiez principalement :

**Plage IP MetalLB** : Dans le script post-installation, ajustez la variable `METALLB_RANGE` selon votre réseau :

```bash
# Réseau 192.168.1.0/24
METALLB_RANGE="192.168.1.200-192.168.1.210"

# Réseau 10.0.0.0/24
METALLB_RANGE="10.0.0.200-10.0.0.210"

# Réseau 172.16.0.0/24
METALLB_RANGE="172.16.0.200-172.16.0.210"
```

**Configuration DNS** : Si vous avez un serveur DNS local :

```bash
# Ajouter dans le script principal après l'activation du DNS
microk8s kubectl -n kube-system edit configmap/coredns

# Ajouter un forwarder vers votre DNS local
# forward . 192.168.1.1 8.8.8.8
```

#### Sélection des Addons

Les addons peuvent être activés/désactivés selon vos besoins :

```bash
# Configuration minimale (développement simple)
ENABLE_DASHBOARD="false"
ENABLE_INGRESS="true"
ENABLE_DNS="true"
ENABLE_STORAGE="true"
ENABLE_REGISTRY="false"
ENABLE_METRICS="false"

# Configuration complète (lab avancé)
ENABLE_DASHBOARD="true"
ENABLE_INGRESS="true"
ENABLE_DNS="true"
ENABLE_STORAGE="true"
ENABLE_REGISTRY="true"
ENABLE_METRICS="true"
```

#### Ajout d'Addons Supplémentaires

Pour ajouter d'autres addons dans le script principal :

```bash
# Exemple : Ajout de l'addon GPU
enable_gpu_addon() {
    log_info "Activation du support GPU..."
    microk8s enable gpu
}

# Exemple : Ajout de Istio
enable_istio_addon() {
    log_info "Activation d'Istio service mesh..."
    microk8s enable istio
}

# Appeler ces fonctions dans enable_addons()
```

### Résolution des Problèmes Courants

#### Problème : Permission Denied

**Symptôme** : Erreurs de permission lors de l'exécution de kubectl

**Solution** :
```bash
# Vérifier l'appartenance au groupe
groups

# Si microk8s n'apparaît pas
sudo usermod -a -G microk8s $USER
newgrp microk8s

# Ou se déconnecter/reconnecter
```

#### Problème : Pods en État Pending

**Symptôme** : Les pods restent en état Pending

**Solutions possibles** :
```bash
# Vérifier les events
kubectl describe pod <nom-du-pod>

# Vérifier le stockage
kubectl get storageclass
kubectl get pv
kubectl get pvc

# Si problème de stockage
microk8s enable hostpath-storage
```

#### Problème : DNS ne Fonctionne Pas

**Symptôme** : Les pods ne peuvent pas résoudre les noms

**Solution** :
```bash
# Vérifier le pod CoreDNS
kubectl get pods -n kube-system | grep dns

# Redémarrer CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Tester la résolution DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

#### Problème : Dashboard Inaccessible

**Symptôme** : Impossible d'accéder au dashboard

**Solution** :
```bash
# Vérifier que le dashboard est actif
kubectl get pods -n kube-system | grep dashboard

# Obtenir le token d'accès
token=$(kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
kubectl -n kube-system describe secret $token

# Port-forward correct
kubectl port-forward -n kube-system service/kubernetes-dashboard 8443:443 --address 0.0.0.0
```

### Scripts Utilitaires Additionnels

#### Script de Backup Rapide

```bash
#!/bin/bash
#
# Backup rapide de la configuration MicroK8s
#

BACKUP_DIR="$HOME/microk8s-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/backup_$TIMESTAMP"

mkdir -p $BACKUP_PATH

# Sauvegarder toutes les ressources
for namespace in $(kubectl get namespaces -o name | cut -d/ -f2); do
    echo "Backup namespace: $namespace"
    kubectl get all -n $namespace -o yaml > "$BACKUP_PATH/${namespace}_resources.yaml"
done

# Sauvegarder les ConfigMaps et Secrets
kubectl get configmaps --all-namespaces -o yaml > "$BACKUP_PATH/configmaps.yaml"
kubectl get secrets --all-namespaces -o yaml > "$BACKUP_PATH/secrets.yaml"

# Sauvegarder les PV et PVC
kubectl get pv -o yaml > "$BACKUP_PATH/persistentvolumes.yaml"
kubectl get pvc --all-namespaces -o yaml > "$BACKUP_PATH/persistentvolumeclaims.yaml"

echo "Backup complété : $BACKUP_PATH"
```

#### Script de Nettoyage des Ressources

```bash
#!/bin/bash
#
# Nettoyage des ressources inutilisées
#

echo "Nettoyage des pods terminés..."
kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces

echo "Nettoyage des pods en erreur..."
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces

echo "Nettoyage des images Docker inutilisées..."
microk8s ctr images prune

echo "Affichage de l'espace récupéré..."
df -h /var/snap/microk8s
```

### Bonnes Pratiques d'Utilisation

#### 1. Sécurité

- **Toujours exécuter les scripts avec sudo** pour éviter les problèmes de permissions
- **Vérifier le contenu des scripts** avant exécution
- **Sauvegarder régulièrement** votre configuration
- **Ne pas exposer le cluster** directement sur Internet sans protection

#### 2. Performance

- **Monitorer les ressources** régulièrement avec le script de santé
- **Nettoyer les ressources inutilisées** périodiquement
- **Limiter les ressources des pods** pour éviter la saturation
- **Utiliser le registry local** pour les images fréquemment utilisées

#### 3. Maintenance

- **Planifier les mises à jour** de MicroK8s
- **Tester les changements** dans un namespace de test
- **Documenter les modifications** apportées aux scripts
- **Versionner vos scripts** dans un repository Git

### Cas d'Usage Typiques

#### Installation Minimale pour Développement

```bash
# Configuration dans le script
ENABLE_DASHBOARD="false"
ENABLE_INGRESS="true"
ENABLE_DNS="true"
ENABLE_STORAGE="true"
ENABLE_REGISTRY="false"
ENABLE_METRICS="false"

# Lancement
sudo ./install-microk8s.sh
```

#### Installation Complète pour Lab d'Apprentissage

```bash
# Configuration dans le script
ENABLE_DASHBOARD="true"
ENABLE_INGRESS="true"
ENABLE_DNS="true"
ENABLE_STORAGE="true"
ENABLE_REGISTRY="true"
ENABLE_METRICS="true"

# Lancement avec post-configuration
sudo ./install-microk8s.sh
sudo ./post-install-config.sh
```

#### Installation pour Environnement de Test CI/CD

```bash
# Ajouter dans le script principal
enable_gitlab_runner() {
    log_info "Configuration pour GitLab Runner..."
    microk8s enable registry
    microk8s enable storage

    # Créer namespace CI/CD
    kubectl create namespace cicd
}

# Utiliser avec ArgoCD ou Flux
```

### Évolution et Maintenance des Scripts

#### Mise à Jour de MicroK8s

Pour mettre à jour MicroK8s via script :

```bash
#!/bin/bash
#
# Script de mise à jour MicroK8s
#

# Sauvegarder la configuration actuelle
./backup-config.sh

# Mettre à jour MicroK8s
sudo snap refresh microk8s --channel=1.29/stable

# Vérifier la mise à jour
microk8s status --wait-ready
microk8s version

# Vérifier les pods
kubectl get pods --all-namespaces
```

#### Ajout de Nouvelles Fonctionnalités

Les scripts sont conçus pour être facilement extensibles :

```bash
# Structure pour ajouter une nouvelle fonctionnalité
add_new_feature() {
    log_info "Installation de la nouvelle fonctionnalité..."

    # Vérifications préalables
    if ! command -v required_tool &> /dev/null; then
        log_warn "Outil requis non trouvé"
        return 1
    fi

    # Installation
    # ...

    # Configuration
    # ...

    # Vérification
    # ...

    log_info "Nouvelle fonctionnalité installée"
}
```

### Conclusion

Ces scripts d'installation automatisée constituent une base solide pour gérer efficacement votre lab MicroK8s. Ils sont conçus pour être :

- **Modulaires** : Chaque fonction peut être utilisée indépendamment
- **Idempotents** : Peuvent être relancés sans risque
- **Documentés** : Commentaires détaillés pour comprendre chaque étape
- **Évolutifs** : Facilement extensibles selon vos besoins

L'automatisation vous permet de vous concentrer sur l'apprentissage et l'expérimentation plutôt que sur la configuration répétitive. N'hésitez pas à adapter ces scripts à vos besoins spécifiques et à les faire évoluer avec votre expérience.

---

*Note : Ces scripts sont fournis à titre d'exemple et doivent être testés dans votre environnement spécifique avant utilisation en production. Assurez-vous toujours de comprendre ce que fait chaque script avant de l'exécuter.*

⏭️
