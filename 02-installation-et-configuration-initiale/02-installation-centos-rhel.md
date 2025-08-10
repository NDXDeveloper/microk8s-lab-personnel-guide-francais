🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Installation sur CentOS/RHEL

## Introduction

CentOS et Red Hat Enterprise Linux (RHEL) sont des distributions Linux orientées entreprise, privilégiant la stabilité et la sécurité. Bien que ces systèmes n'incluent pas Snap par défaut comme Ubuntu, l'installation de MicroK8s reste accessible avec quelques étapes supplémentaires. Cette section vous guidera à travers le processus complet, en tenant compte des particularités de l'écosystème Red Hat.

**Note importante** : Depuis fin 2021, CentOS a évolué vers CentOS Stream (version rolling release). Les instructions de ce guide s'appliquent à :
- CentOS 7 (maintenance jusqu'en 2024)
- CentOS Stream 8 et 9
- RHEL 7, 8 et 9
- Rocky Linux et AlmaLinux (alternatives à CentOS)

## Prérequis spécifiques à CentOS/RHEL

### Vérification de la version et de la compatibilité

```bash
# Identifier votre distribution et version
cat /etc/redhat-release

# Ou pour plus de détails
hostnamectl

# Vérifier l'architecture (doit être x86_64 ou aarch64)
uname -m
```

**Versions minimales supportées** :
- CentOS/RHEL 7.6 ou supérieur
- CentOS Stream 8 ou supérieur
- Rocky Linux 8 ou supérieur
- AlmaLinux 8 ou supérieur

### Vérification des ressources système

```bash
# Mémoire disponible (minimum 2GB, recommandé 4GB+)
free -h

# Espace disque disponible (minimum 20GB)
df -h /

# Nombre de processeurs (minimum 1, recommandé 2+)
nproc

# Version du kernel (doit être 3.10 ou supérieur)
uname -r
```

### Configuration SELinux

SELinux (Security-Enhanced Linux) est activé par défaut sur CentOS/RHEL et peut interférer avec MicroK8s. Vérifiez son état :

```bash
# Vérifier le statut de SELinux
getenforce

# Voir la configuration détaillée
sestatus
```

Vous avez trois options pour gérer SELinux avec MicroK8s :

**Option 1 : Mode Permissif (Recommandé pour les labs)**
```bash
# Passer temporairement en mode permissif
sudo setenforce 0

# Pour rendre le changement permanent
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**Option 2 : Désactiver SELinux (Plus simple, moins sécurisé)**
```bash
# Désactiver complètement SELinux
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# Redémarrage nécessaire
sudo reboot
```

**Option 3 : Configurer les politiques SELinux (Avancé)**
Si vous souhaitez maintenir SELinux en mode enforcing, vous devrez créer des politiques personnalisées après l'installation (couvert en annexe).

## Installation des dépendances

### Mise à jour du système

```bash
# Pour CentOS/RHEL 7
sudo yum update -y

# Pour CentOS/RHEL 8+ et Stream
sudo dnf update -y
```

### Installation des outils essentiels

```bash
# Pour CentOS/RHEL 7
sudo yum install -y epel-release
sudo yum install -y wget curl git vim net-tools

# Pour CentOS/RHEL 8+
sudo dnf install -y epel-release
sudo dnf install -y wget curl git vim net-tools
```

### Configuration du firewall

CentOS/RHEL utilise firewalld par défaut. Préparez les règles nécessaires :

```bash
# Vérifier si firewalld est actif
sudo systemctl status firewalld

# Si actif, ouvrir les ports nécessaires pour MicroK8s
sudo firewall-cmd --permanent --zone=public --add-port=16443/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10250/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10255/tcp
sudo firewall-cmd --permanent --zone=public --add-port=25000/tcp
sudo firewall-cmd --permanent --zone=public --add-port=12345/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10257/tcp
sudo firewall-cmd --permanent --zone=public --add-port=10259/tcp

# Autoriser le forwarding pour les pods
sudo firewall-cmd --permanent --zone=public --add-masquerade

# Recharger la configuration
sudo firewall-cmd --reload

# Vérifier les règles appliquées
sudo firewall-cmd --list-all
```

## Installation de Snap sur CentOS/RHEL

Snap n'est pas disponible dans les dépôts officiels Red Hat. L'installation varie selon la version.

### Installation sur CentOS/RHEL 7

```bash
# Installer snapd depuis EPEL
sudo yum install -y snapd

# Activer et démarrer le service snapd
sudo systemctl enable --now snapd.socket

# Créer le lien symbolique pour le support classic
sudo ln -s /var/lib/snapd/snap /snap

# Attendre que snapd soit complètement initialisé
sudo systemctl restart snapd
sleep 10

# Installer le core snap
sudo snap install core
sudo snap refresh core
```

### Installation sur CentOS/RHEL 8+

```bash
# Installer snapd depuis EPEL
sudo dnf install -y snapd

# Activer le service snapd
sudo systemctl enable --now snapd.socket

# Créer le lien symbolique nécessaire
sudo ln -s /var/lib/snapd/snap /snap

# Redémarrer pour s'assurer que tout est chargé
sudo systemctl restart snapd

# Attendre l'initialisation
sleep 10

# Installer et rafraîchir core
sudo snap install core
sudo snap refresh core
```

### Installation sur CentOS Stream 9

CentOS Stream 9 nécessite quelques étapes supplémentaires :

```bash
# Activer le dépôt EPEL Next
sudo dnf install -y epel-release epel-next-release

# Installer snapd
sudo dnf install -y snapd

# Configurer et démarrer snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap

# Important : Se déconnecter et se reconnecter pour mettre à jour le PATH
echo "Please logout and login again to update PATH"
```

**Important** : Après l'installation de snapd, déconnectez-vous et reconnectez-vous pour que les changements de PATH prennent effet :

```bash
# Vérifier que snap est dans le PATH
which snap

# Si non trouvé, ajouter manuellement
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

## Installation de MicroK8s

### Désactivation temporaire de la protection du kernel

Sur CentOS/RHEL, certaines protections kernel peuvent interférer :

```bash
# Configurer les paramètres kernel pour Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

# Appliquer les changements
sudo sysctl --system

# Désactiver le swap
sudo swapoff -a

# Commenter les lignes swap dans fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Installation de MicroK8s via Snap

```bash
# Installer MicroK8s
sudo snap install microk8s --classic --channel=1.31/stable

# Vérifier l'installation
snap list microk8s
```

Si vous rencontrez l'erreur "too early for operation", attendez et réessayez :

```bash
# Attendre que snapd soit prêt
sudo snap wait system seed.loaded

# Réessayer l'installation
sudo snap install microk8s --classic
```

### Alternative : Installation sans Snap

Si Snap pose problème, vous pouvez utiliser l'installation alternative via les binaires :

```bash
# Télécharger le script d'installation alternatif
curl -LO https://raw.githubusercontent.com/ubuntu/microk8s/master/installer/mk8s-installer.sh

# Rendre exécutable et lancer
chmod +x mk8s-installer.sh
sudo ./mk8s-installer.sh
```

## Configuration post-installation

### Gestion des groupes et permissions

```bash
# Créer le groupe microk8s s'il n'existe pas
sudo groupadd -f microk8s

# Ajouter votre utilisateur au groupe
sudo usermod -a -G microk8s $USER

# Donner les permissions au groupe
sudo chown -R root:microk8s /var/snap/microk8s/current/

# Créer le répertoire .kube
mkdir -p ~/.kube

# Se reconnecter pour appliquer les changements de groupe
su - $USER
```

### Configuration spécifique pour CentOS/RHEL

#### Ajustement de cgroup

CentOS/RHEL peut utiliser cgroup v1 ou v2. MicroK8s fonctionne mieux avec v1 :

```bash
# Vérifier la version de cgroup
mount | grep cgroup

# Si cgroup2 est utilisé, forcer cgroup v1
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"

# Redémarrer si nécessaire
# sudo reboot
```

#### Configuration du résolveur DNS

```bash
# S'assurer que systemd-resolved n'interfère pas
sudo systemctl status systemd-resolved

# Si actif sur RHEL 8+, le configurer correctement
sudo mkdir -p /etc/systemd/resolved.conf.d/
cat <<EOF | sudo tee /etc/systemd/resolved.conf.d/microk8s.conf
[Resolve]
DNSStubListener=no
EOF

sudo systemctl restart systemd-resolved
```

### Vérification de l'installation

```bash
# Attendre que MicroK8s soit prêt
microk8s status --wait-ready

# Vérifier les composants
microk8s inspect

# Tester kubectl
microk8s kubectl get nodes

# Vérifier les pods système
microk8s kubectl get pods --all-namespaces
```

## Gestion des problèmes spécifiques à CentOS/RHEL

### Problème : "cannot communicate with server"

**Cause** : SELinux ou firewalld bloque les communications

**Solution** :
```bash
# Vérifier les logs SELinux
sudo ausearch -m avc -ts recent

# Si SELinux est le problème
sudo setenforce 0

# Vérifier firewalld
sudo firewall-cmd --list-all
sudo systemctl stop firewalld  # Temporairement pour tester
```

### Problème : Snap ne fonctionne pas correctement

**Symptômes** : Erreurs lors de l'installation de snaps

**Solutions** :
```bash
# Réinitialiser snapd
sudo systemctl stop snapd
sudo systemctl stop snapd.socket
sudo rm -rf /var/lib/snapd/cache/*
sudo systemctl start snapd.socket
sudo systemctl start snapd

# Vérifier les montages
mount | grep snapd

# Si nécessaire, remonter
sudo mount -t squashfs /var/lib/snapd/snaps/core_*.snap /snap/core/current
```

### Problème : Erreurs de cgroup

**Symptôme** : Pods qui ne démarrent pas, erreurs liées à cgroup

**Solution** :
```bash
# Vérifier la configuration de cgroup
cat /proc/cgroups

# Installer les outils cgroup si manquants
sudo yum install -y libcgroup-tools  # CentOS 7
sudo dnf install -y cgroup-tools     # CentOS 8+

# Redémarrer les services cgroup
sudo systemctl restart cgconfig
sudo systemctl restart cgred
```

### Problème : Performance réseau dégradée

**Cause** : NetworkManager peut interférer avec la configuration réseau de Kubernetes

**Solution** :
```bash
# Configurer NetworkManager pour ignorer les interfaces de MicroK8s
cat <<EOF | sudo tee /etc/NetworkManager/conf.d/calico.conf
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico
EOF

sudo systemctl restart NetworkManager
```

## Optimisations pour CentOS/RHEL

### Tuning pour les performances

```bash
# Optimisations kernel pour les conteneurs
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-tuning.conf
# Réseau
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0

# Mémoire
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10

# Fichiers
fs.inotify.max_user_watches = 1048576
fs.inotify.max_user_instances = 8192
EOF

sudo sysctl -p /etc/sysctl.d/99-kubernetes-tuning.conf
```

### Configuration du journal système

Pour éviter que les logs saturent le disque :

```bash
# Configurer journald
sudo mkdir -p /etc/systemd/journald.conf.d/
cat <<EOF | sudo tee /etc/systemd/journald.conf.d/microk8s.conf
[Journal]
SystemMaxUse=2G
SystemMaxFileSize=200M
MaxRetentionSec=2week
EOF

sudo systemctl restart systemd-journald
```

## Validation finale sur CentOS/RHEL

### Tests de validation complets

```bash
# Test 1 : Cluster status
microk8s status

# Test 2 : Déploiement test
microk8s kubectl create deployment hello --image=hello-world
microk8s kubectl wait --for=condition=complete job/hello --timeout=60s
microk8s kubectl logs deployment/hello
microk8s kubectl delete deployment hello

# Test 3 : Réseau pod-to-pod
microk8s kubectl run test-net --image=busybox --rm -it --restart=Never -- ping -c 3 8.8.8.8

# Test 4 : DNS interne
microk8s kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

### Script de vérification automatique

Créez un script pour valider rapidement votre installation :

```bash
cat <<'EOF' > ~/check-microk8s.sh
#!/bin/bash

echo "=== MicroK8s Installation Check for CentOS/RHEL ==="

# Couleurs pour l'output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

# Fonction de vérification
check() {
    if eval "$2"; then
        echo -e "${GREEN}✓${NC} $1"
        return 0
    else
        echo -e "${RED}✗${NC} $1"
        return 1
    fi
}

# Vérifications
check "SELinux en mode permissif ou désactivé" "[[ $(getenforce) != 'Enforcing' ]]"
check "Snapd installé et actif" "systemctl is-active snapd -q"
check "MicroK8s installé" "snap list microk8s &>/dev/null"
check "MicroK8s en cours d'exécution" "microk8s status | grep -q 'microk8s is running'"
check "Kubectl accessible" "microk8s kubectl version &>/dev/null"
check "Node prêt" "microk8s kubectl get nodes | grep -q Ready"
check "Pods système en cours d'exécution" "[[ $(microk8s kubectl get pods -A --field-selector=status.phase!=Running 2>/dev/null | wc -l) -le 1 ]]"
check "Firewall configuré" "firewall-cmd --list-ports | grep -q 16443"

echo "=== Vérification terminée ==="
EOF

chmod +x ~/check-microk8s.sh
~/check-microk8s.sh
```

## Maintenance sur CentOS/RHEL

### Mises à jour de sécurité

```bash
# Mettre à jour le système sans toucher à MicroK8s
sudo yum update -y --exclude=snapd  # CentOS 7
sudo dnf update -y --exclude=snapd  # CentOS 8+

# Mettre à jour MicroK8s séparément
sudo snap refresh microk8s
```

### Logs et diagnostic

```bash
# Logs MicroK8s
sudo journalctl -u snap.microk8s.daemon-kubelite -f

# Logs SELinux liés à MicroK8s
sudo ausearch -m avc -c microk8s

# Diagnostic complet
microk8s inspect | tee microk8s-inspect-$(date +%Y%m%d).log
```

## Points de vérification finale

Avant de passer à la section suivante, confirmez que :

- ✅ MicroK8s est installé et `microk8s status` indique "running"
- ✅ SELinux est en mode permissif ou désactivé (ou correctement configuré)
- ✅ Le firewall autorise les ports nécessaires
- ✅ Vous pouvez exécuter `microk8s kubectl` sans erreur
- ✅ Un déploiement test fonctionne correctement
- ✅ Les logs ne montrent pas d'erreurs critiques

## Différences clés avec Ubuntu/Debian

Pour ceux venant d'Ubuntu/Debian, voici les principales différences sur CentOS/RHEL :

1. **SELinux** : Politique de sécurité obligatoire additionnelle
2. **Firewalld** : Système de firewall différent d'UFW
3. **Snap** : Non natif, nécessite EPEL
4. **SystemD** : Configuration parfois différente
5. **Cgroups** : Peut nécessiter des ajustements
6. **NetworkManager** : Gestion réseau différente

## Prochaines étapes

Avec MicroK8s opérationnel sur CentOS/RHEL, vous pouvez maintenant :

1. Configurer kubectl avec des alias pratiques (section 2.6)
2. Explorer et activer les addons nécessaires
3. Adapter la configuration réseau à votre environnement d'entreprise
4. Implémenter les politiques de sécurité appropriées

La robustesse de CentOS/RHEL combinée à la simplicité de MicroK8s offre une excellente plateforme pour un lab Kubernetes professionnel.

⏭️
