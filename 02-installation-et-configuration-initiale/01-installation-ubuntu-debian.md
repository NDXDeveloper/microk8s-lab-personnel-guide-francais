🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.1 Installation sur Ubuntu/Debian

## Introduction

Ubuntu et Debian représentent les plateformes de choix pour MicroK8s, offrant une intégration native via le système de packages Snap. Ubuntu, en particulier, est développé par Canonical, la même société qui maintient MicroK8s, garantissant ainsi une compatibilité et un support optimaux. Cette section vous guidera pas à pas dans l'installation, depuis la préparation du système jusqu'à la validation finale.

## Vérification des prérequis système

### Vérifier la version du système d'exploitation

Avant toute installation, confirmez que votre système est supporté. Ouvrez un terminal et exécutez :

```bash
# Pour Ubuntu
lsb_release -a

# Pour Debian
cat /etc/debian_version
```

**Versions minimales supportées** :
- Ubuntu : 18.04 LTS (Bionic Beaver) ou supérieur
- Debian : 10 (Buster) ou supérieur

Pour une expérience optimale, Ubuntu 22.04 LTS ou Debian 12 sont recommandés car ils incluent des optimisations kernel pour les conteneurs.

### Vérifier les ressources disponibles

Assurez-vous que votre système dispose des ressources nécessaires :

```bash
# Vérifier la mémoire disponible (minimum 2GB, recommandé 4GB+)
free -h

# Vérifier l'espace disque (minimum 20GB disponible)
df -h /

# Vérifier le nombre de CPU (minimum 1, recommandé 2+)
nproc

# Vérifier l'architecture (doit être x86_64 ou arm64)
uname -m
```

### Vérifier les ports réseau

MicroK8s utilise plusieurs ports qui doivent être disponibles :

```bash
# Vérifier si les ports critiques sont libres
sudo ss -tuln | grep -E ':(16443|10250|10255|25000)'
```

Si ces commandes retournent des résultats, les ports sont déjà utilisés et vous devrez soit arrêter les services en conflit, soit configurer MicroK8s pour utiliser d'autres ports (configuration avancée).

## Installation de Snap (si nécessaire)

### Sur Ubuntu

Ubuntu 16.04 et versions ultérieures incluent Snap par défaut. Vérifiez sa présence :

```bash
snap --version
```

Si Snap n'est pas installé (rare sur Ubuntu), installez-le :

```bash
sudo apt update
sudo apt install snapd
```

### Sur Debian

Debian nécessite généralement l'installation manuelle de Snap :

```bash
# Mettre à jour le système
sudo apt update
sudo apt upgrade -y

# Installer snapd
sudo apt install snapd

# Installer le core snap
sudo snap install core

# Redémarrer le service snapd
sudo systemctl restart snapd
```

**Note importante pour Debian** : Après l'installation de snapd, vous devrez peut-être vous déconnecter et vous reconnecter, ou sourcer votre profil pour que les commandes snap soient disponibles dans votre PATH :

```bash
# Ajouter snap au PATH si nécessaire
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

## Installation de MicroK8s

### Méthode standard (canal stable)

La méthode la plus simple et recommandée pour les débutants :

```bash
# Installer MicroK8s depuis le canal stable
sudo snap install microk8s --classic --channel=1.31/stable
```

**Explication des options** :
- `--classic` : Permet à MicroK8s d'accéder aux ressources système nécessaires (réseau, stockage)
- `--channel=1.31/stable` : Spécifie la version Kubernetes 1.31 depuis le canal stable

### Choisir la bonne version

MicroK8s propose plusieurs canaux de distribution :

```bash
# Voir les canaux disponibles
snap info microk8s
```

**Canaux principaux** :
- `1.31/stable` : Version stable actuelle, recommandée pour la production
- `1.31/candidate` : Version en phase de test final
- `1.31/beta` : Version beta avec nouvelles fonctionnalités
- `1.31/edge` : Version de développement quotidienne
- `latest/stable` : Dernière version stable (suit automatiquement les mises à jour majeures)

Pour un lab personnel, le canal stable de la dernière version est généralement le meilleur choix :

```bash
# Installation avec la dernière version stable
sudo snap install microk8s --classic
```

### Installation hors ligne (optionnel)

Si vous travaillez dans un environnement sans connexion Internet, vous pouvez télécharger le snap depuis une machine connectée :

```bash
# Sur une machine avec Internet
snap download microk8s --channel=1.31/stable

# Cela télécharge deux fichiers :
# - microk8s_[version].snap
# - microk8s_[version].assert
```

Transférez ces fichiers sur votre machine cible et installez :

```bash
# Installation depuis les fichiers locaux
sudo snap ack microk8s_*.assert
sudo snap install microk8s_*.snap --classic
```

## Configuration post-installation

### Ajouter votre utilisateur au groupe microk8s

Par défaut, toutes les commandes MicroK8s nécessitent sudo. Pour éviter cela :

```bash
# Ajouter l'utilisateur actuel au groupe microk8s
sudo usermod -a -G microk8s $USER

# Créer le répertoire .kube si nécessaire
mkdir -p ~/.kube

# Donner la propriété à l'utilisateur
sudo chown -R $USER ~/.kube
```

**Important** : Vous devez vous déconnecter et vous reconnecter pour que les changements de groupe prennent effet :

```bash
# Option 1 : Se reconnecter complètement
logout
# Puis se reconnecter

# Option 2 : Recharger le groupe dans la session actuelle (Linux seulement)
newgrp microk8s
```

### Vérifier le statut de MicroK8s

Après la reconnexion, vérifiez que MicroK8s fonctionne :

```bash
# Vérifier le statut général
microk8s status --wait-ready

# Afficher les informations détaillées
microk8s inspect
```

La commande `status` devrait afficher quelque chose comme :

```
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
```

### Configuration du firewall (UFW)

Si vous utilisez UFW (Uncomplicated Firewall) sur Ubuntu :

```bash
# Vérifier si UFW est actif
sudo ufw status

# Si actif, autoriser les connexions pour MicroK8s
sudo ufw allow in on cni0
sudo ufw allow out on cni0
sudo ufw default allow routed

# Pour l'accès à l'API depuis l'extérieur (optionnel, pour accès distant)
sudo ufw allow 16443/tcp
```

Pour Debian avec iptables, MicroK8s gère généralement ses propres règles automatiquement.

### Optimisations système recommandées

Pour de meilleures performances, appliquez ces optimisations :

```bash
# Désactiver swap (recommandé pour Kubernetes)
sudo swapoff -a

# Pour rendre le changement permanent
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Configurer les paramètres kernel pour Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Appliquer les changements
sudo sysctl --system
```

## Validation de l'installation

### Tests de base

Effectuez ces tests pour confirmer que votre installation est fonctionnelle :

```bash
# Test 1 : Vérifier les nodes
microk8s kubectl get nodes

# Devrait afficher quelque chose comme :
# NAME        STATUS   ROLES    AGE   VERSION
# ubuntu-vm   Ready    <none>   1m    v1.31.0

# Test 2 : Vérifier les pods système
microk8s kubectl get pods --all-namespaces

# Test 3 : Vérifier les services
microk8s kubectl get services --all-namespaces
```

### Déploiement de test

Validez que vous pouvez déployer des applications :

```bash
# Créer un déploiement nginx simple
microk8s kubectl create deployment nginx --image=nginx

# Vérifier que le pod est en cours d'exécution
microk8s kubectl get pods

# Attendre que le pod soit Ready
microk8s kubectl wait --for=condition=ready pod -l app=nginx --timeout=60s

# Nettoyer
microk8s kubectl delete deployment nginx
```

### Vérification des logs

En cas de problème, consultez les logs :

```bash
# Logs du service MicroK8s
sudo journalctl -u snap.microk8s.daemon-kubelite -n 50

# Inspection complète (génère un rapport)
microk8s inspect

# Logs d'un composant spécifique
microk8s kubectl logs -n kube-system deployment/coredns
```

## Gestion du service MicroK8s

### Commandes de contrôle essentielles

```bash
# Arrêter MicroK8s
microk8s stop

# Démarrer MicroK8s
microk8s start

# Redémarrer MicroK8s
microk8s restart

# Voir la consommation de ressources
microk8s kubectl top nodes
```

### Configuration de démarrage automatique

Par défaut, MicroK8s démarre automatiquement au boot. Pour modifier ce comportement :

```bash
# Désactiver le démarrage automatique
sudo snap stop --disable microk8s

# Réactiver le démarrage automatique
sudo snap start --enable microk8s
```

## Résolution des problèmes courants

### Problème : "Permission denied" après installation

**Symptôme** : Erreurs de permission même après ajout au groupe microk8s

**Solution** :
```bash
# Vérifier l'appartenance au groupe
groups

# Si microk8s n'apparaît pas, se reconnecter complètement
# Si le problème persiste, vérifier les permissions
ls -la /var/snap/microk8s/current/credentials/
sudo chown root:microk8s /var/snap/microk8s/current/credentials/
sudo chmod 660 /var/snap/microk8s/current/credentials/*
```

### Problème : MicroK8s ne démarre pas

**Symptôme** : `microk8s status` indique "not running"

**Solutions à essayer** :
```bash
# Vérifier les logs système
sudo journalctl -xe | grep microk8s

# Réinitialiser MicroK8s
microk8s reset

# Vérifier l'espace disque
df -h /var/snap/microk8s

# Vérifier les conflits de ports
sudo netstat -tulpn | grep -E ':(16443|10250)'
```

### Problème : Pods en état "Pending" ou "ContainerCreating"

**Symptôme** : Les pods ne démarrent pas correctement

**Solutions** :
```bash
# Vérifier les événements
microk8s kubectl describe pod [nom-du-pod]

# Vérifier le réseau
microk8s inspect

# Redémarrer le DNS
microk8s kubectl rollout restart -n kube-system deployment/coredns
```

### Problème : Connectivité réseau

**Symptôme** : Pas d'accès Internet depuis les pods

**Solution** :
```bash
# Vérifier la configuration DNS
microk8s kubectl get configmap -n kube-system coredns -o yaml

# Test de connectivité
microk8s kubectl run test-pod --image=busybox --rm -it --restart=Never -- ping 8.8.8.8

# Vérifier les règles iptables
sudo iptables -L -n -v
```

## Mises à jour et maintenance

### Mise à jour de MicroK8s

MicroK8s se met à jour automatiquement par défaut. Pour gérer manuellement :

```bash
# Vérifier les mises à jour disponibles
sudo snap refresh --list

# Mettre à jour manuellement
sudo snap refresh microk8s

# Changer de canal (ex: passer en version beta)
sudo snap refresh microk8s --channel=1.31/beta
```

### Sauvegarde de la configuration

Avant toute mise à jour majeure :

```bash
# Sauvegarder les certificats et configurations
sudo tar -czf microk8s-backup-$(date +%Y%m%d).tar.gz \
  /var/snap/microk8s/current/credentials \
  /var/snap/microk8s/current/certs \
  /var/snap/microk8s/current/args

# Exporter les ressources Kubernetes
microk8s kubectl get all --all-namespaces -o yaml > cluster-resources-$(date +%Y%m%d).yaml
```

## Points de vérification finale

Avant de passer à la section suivante, assurez-vous que :

- ✅ MicroK8s est installé et affiche "is running" avec `microk8s status`
- ✅ Vous pouvez exécuter des commandes sans sudo (après ajout au groupe)
- ✅ `microk8s kubectl get nodes` montre votre node en état "Ready"
- ✅ Vous pouvez créer et supprimer un déploiement de test
- ✅ Les logs ne montrent pas d'erreurs critiques
- ✅ Vous comprenez comment démarrer/arrêter MicroK8s

## Prochaines étapes

Avec MicroK8s installé et fonctionnel sur votre système Ubuntu/Debian, vous êtes prêt à :

1. Configurer kubectl pour une utilisation plus pratique (section 2.6)
2. Explorer les addons disponibles pour étendre les fonctionnalités
3. Configurer le réseau et le domaine pour l'accès externe
4. Commencer à déployer vos premières applications

La base solide que vous venez d'établir servira de fondation pour toutes les configurations avancées que nous aborderons dans les sections suivantes.

⏭️
