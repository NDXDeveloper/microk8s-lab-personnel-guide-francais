🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Vérification de l'installation

## Introduction

Après avoir installé MicroK8s sur votre système d'exploitation, il est crucial de vérifier que tous les composants fonctionnent correctement avant de commencer à déployer des applications. Cette section vous guide à travers une série de vérifications systématiques pour confirmer que votre installation est opérationnelle et prête à l'emploi. Ces tests vous permettront également de vous familiariser avec les commandes de base et de diagnostiquer rapidement tout problème éventuel.

## Vérifications de base

### État général de MicroK8s

La première vérification consiste à s'assurer que MicroK8s est actif et fonctionnel :

```bash
# Vérifier le statut de MicroK8s
microk8s status

# Sortie attendue :
# microk8s is running
# high-availability: no
#   datastore master nodes: 127.0.0.1:19001
#   datastore standby nodes: none
```

**Interprétation des résultats** :
- `is running` : MicroK8s est actif et prêt
- `is not running` : MicroK8s est arrêté, utilisez `microk8s start`
- `high-availability: no` : Cluster single-node (normal pour un lab)
- `datastore master nodes` : Endpoint de la base de données etcd

Si MicroK8s n'est pas en cours d'exécution :
```bash
# Démarrer MicroK8s
microk8s start

# Attendre qu'il soit prêt (peut prendre 1-2 minutes)
microk8s status --wait-ready
```

### Version et composants installés

```bash
# Vérifier la version de MicroK8s
microk8s version

# Sortie exemple :
# MicroK8s v1.31.0 revision 1234

# Vérifier la version de Kubernetes
microk8s kubectl version

# Sortie attendue :
# Client Version: v1.31.0
# Server Version: v1.31.0
```

**Important** : Les versions Client et Server doivent être compatibles (idéalement identiques).

### Vérification des addons

```bash
# Lister tous les addons et leur statut
microk8s status --format short

# Sortie exemple :
# microk8s is running
# addons:
#   enabled:
#     dns                  # CoreDNS pour la résolution de noms
#     ha-cluster           # Cluster haute disponibilité
#     storage              # Stockage local
#   disabled:
#     dashboard            # Dashboard Kubernetes
#     ingress              # Contrôleur Ingress NGINX
#     metrics-server       # Métriques des ressources
```

## Vérification du cluster Kubernetes

### État des nodes

```bash
# Vérifier l'état du node
microk8s kubectl get nodes

# Sortie attendue :
# NAME        STATUS   ROLES    AGE   VERSION
# hostname    Ready    <none>   10m   v1.31.0
```

**Statuts possibles** :
- `Ready` : Le node est opérationnel
- `NotReady` : Problème avec kubelet ou le réseau
- `SchedulingDisabled` : Le node n'accepte pas de nouveaux pods

Pour plus de détails sur un node :
```bash
# Informations détaillées
microk8s kubectl describe node

# Points à vérifier :
# - Conditions (toutes doivent être "False" sauf Ready qui doit être "True")
# - Allocated resources (CPU, mémoire, pods)
# - System Info (OS, kernel, runtime)
```

### Vérification des namespaces

```bash
# Lister les namespaces
microk8s kubectl get namespaces

# Namespaces par défaut attendus :
# NAME              STATUS   AGE
# default           Active   10m
# kube-node-lease   Active   10m
# kube-public       Active   10m
# kube-system       Active   10m
```

**Rôle de chaque namespace** :
- `default` : Namespace par défaut pour vos applications
- `kube-system` : Composants système de Kubernetes
- `kube-public` : Ressources publiques accessibles à tous
- `kube-node-lease` : Gestion des baux de nodes pour la haute disponibilité

### Pods système

```bash
# Vérifier les pods système
microk8s kubectl get pods --all-namespaces

# Ou version courte
microk8s kubectl get pods -A

# Sortie typique (avec DNS activé) :
# NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
# kube-system   calico-node-xxxxx                        1/1     Running   0          10m
# kube-system   calico-kube-controllers-xxxxxxxxx-xxxxx  1/1     Running   0          10m
# kube-system   coredns-xxxxxxxxx-xxxxx                  1/1     Running   0          10m
```

**Pods système essentiels** :
- `calico-*` : Réseau et politiques réseau
- `coredns-*` : Résolution DNS interne
- Tous doivent être en statut `Running` avec `1/1` Ready

Si des pods ne sont pas Running :
```bash
# Voir les détails d'un pod problématique
microk8s kubectl describe pod <nom-du-pod> -n <namespace>

# Voir les logs d'un pod
microk8s kubectl logs <nom-du-pod> -n <namespace>
```

## Tests fonctionnels

### Test de création de ressources

Vérifiez que vous pouvez créer et gérer des ressources :

```bash
# Créer un déploiement de test
microk8s kubectl create deployment test-nginx --image=nginx:alpine

# Vérifier le déploiement
microk8s kubectl get deployments

# Vérifier le pod créé
microk8s kubectl get pods

# Le pod doit passer de ContainerCreating à Running
# NAME                          READY   STATUS    RESTARTS   AGE
# test-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

### Test de connectivité réseau

```bash
# Exposer le déploiement de test
microk8s kubectl expose deployment test-nginx --port=80 --type=ClusterIP

# Obtenir l'IP du service
SERVICE_IP=$(microk8s kubectl get service test-nginx -o jsonpath='{.spec.clusterIP}')
echo "Service IP: $SERVICE_IP"

# Tester la connectivité (depuis le node)
curl -I $SERVICE_IP

# Réponse attendue :
# HTTP/1.1 200 OK
# Server: nginx/...
```

### Test DNS interne

```bash
# Créer un pod pour tester le DNS
microk8s kubectl run test-dns --image=busybox:latest --rm -it --restart=Never -- nslookup kubernetes.default

# Sortie attendue :
# Server:    10.152.183.10
# Address:   10.152.183.10:53
#
# Name:      kubernetes.default.svc.cluster.local
# Address:   10.152.183.1
```

### Nettoyage des tests

```bash
# Supprimer les ressources de test
microk8s kubectl delete deployment test-nginx
microk8s kubectl delete service test-nginx
```

## Vérification des ressources système

### Utilisation CPU et mémoire

```bash
# Si metrics-server est activé
microk8s enable metrics-server

# Attendre 1-2 minutes puis :
microk8s kubectl top nodes

# Sortie exemple :
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# hostname   250m         12%    1824Mi          45%

# Voir l'utilisation par pod
microk8s kubectl top pods -A
```

### Espace disque

```bash
# Vérifier l'espace utilisé par MicroK8s
du -sh /var/snap/microk8s/common/

# Vérifier l'espace disponible
df -h /var/snap/microk8s/

# Points d'attention :
# - Au moins 5GB libres recommandés
# - Les images de conteneurs peuvent consommer beaucoup d'espace
```

### Vérification des logs

```bash
# Logs du service MicroK8s (systemd)
sudo journalctl -u snap.microk8s.daemon-kubelite -n 50

# Logs via kubectl
microk8s kubectl logs -n kube-system deployment/coredns

# Événements du cluster
microk8s kubectl get events -A --sort-by='.lastTimestamp'
```

## Diagnostic approfondi avec inspect

MicroK8s fournit un outil de diagnostic complet :

```bash
# Lancer une inspection complète
microk8s inspect

# L'inspection vérifie :
# - Services en cours d'exécution
# - Connectivité réseau
# - Configuration DNS
# - Certificats
# - Permissions fichiers
# - Ports ouverts
```

**Analyse du rapport d'inspection** :

Le rapport est sauvegardé dans un tarball. Pour l'examiner :
```bash
# Le fichier est créé avec un nom comme :
# inspection-report-YYYYMMDD_HHMMSS.tar.gz

# Extraire et examiner
tar -xzf inspection-report-*.tar.gz
cd inspection-report-*/

# Fichiers importants :
# - microk8s-inspect.log : Résumé des vérifications
# - systemctl.log : État des services
# - network.log : Configuration réseau
# - errors.log : Erreurs détectées
```

## Vérifications spécifiques par OS

### Linux (Ubuntu/Debian/CentOS)

```bash
# Vérifier les groupes utilisateur
groups | grep microk8s

# Vérifier les services systemd
systemctl status snap.microk8s.daemon-kubelite

# Vérifier les montages
mount | grep microk8s

# Vérifier SELinux (CentOS/RHEL)
getenforce
```

### Windows (WSL2)

```powershell
# Depuis PowerShell, vérifier WSL2
wsl --list --verbose

# Vérifier les ressources WSL2
wsl -d Ubuntu-22.04 -e sh -c "free -h && df -h /"

# Tester l'accès depuis Windows
wsl -d Ubuntu-22.04 -e microk8s status
```

### macOS (Multipass)

```bash
# Vérifier la VM Multipass
multipass list

# Vérifier les ressources de la VM
multipass info microk8s-vm

# Tester l'accès depuis macOS
multipass exec microk8s-vm -- microk8s status
```

## Tableau de diagnostic rapide

| Symptôme | Commande de vérification | Solution probable |
|----------|-------------------------|-------------------|
| MicroK8s ne démarre pas | `microk8s status` | `sudo snap restart microk8s` |
| Node NotReady | `microk8s kubectl get nodes` | Vérifier kubelet : `sudo journalctl -u snap.microk8s.daemon-kubelite` |
| Pods en Pending | `microk8s kubectl describe pod <nom>` | Vérifier les ressources disponibles |
| Pods en CrashLoopBackOff | `microk8s kubectl logs <pod>` | Examiner les logs pour l'erreur |
| DNS ne fonctionne pas | `microk8s kubectl get pods -n kube-system` | Redémarrer CoreDNS |
| Pas d'accès Internet | Test avec `curl` depuis un pod | Vérifier la configuration réseau |
| Certificats expirés | `microk8s inspect` | `microk8s refresh-certs` |

## Script de vérification automatisée

Créez un script pour automatiser les vérifications :

```bash
cat << 'EOF' > ~/verify-microk8s.sh
#!/bin/bash

# Couleurs pour l'affichage
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo "======================================"
echo "    MicroK8s Installation Checker    "
echo "======================================"
echo ""

# Fonction de vérification
check() {
    local description=$1
    local command=$2

    echo -n "Checking $description... "

    if eval $command &>/dev/null; then
        echo -e "${GREEN}✓ OK${NC}"
        return 0
    else
        echo -e "${RED}✗ FAILED${NC}"
        echo "  Command: $command"
        return 1
    fi
}

# Compteur d'erreurs
errors=0

# Vérifications
check "MicroK8s is installed" "which microk8s" || ((errors++))
check "MicroK8s is running" "microk8s status | grep -q 'is running'" || ((errors++))
check "Kubectl works" "microk8s kubectl version --short" || ((errors++))
check "Node is ready" "microk8s kubectl get nodes | grep -q Ready" || ((errors++))
check "System pods are running" "[ $(microk8s kubectl get pods -n kube-system --field-selector=status.phase!=Running 2>/dev/null | wc -l) -le 1 ]" || ((errors++))
check "DNS addon is enabled" "microk8s status | grep -q 'dns.*enabled'" || ((errors++))
check "Can create deployments" "microk8s kubectl auth can-i create deployments" || ((errors++))
check "Network connectivity" "microk8s kubectl run test-connectivity --image=busybox --rm -it --restart=Never -- echo 'OK' 2>/dev/null | grep -q OK" || ((errors++))

echo ""
echo "======================================"
if [ $errors -eq 0 ]; then
    echo -e "${GREEN}All checks passed! MicroK8s is ready.${NC}"
else
    echo -e "${YELLOW}$errors check(s) failed. Please review the errors above.${NC}"
    echo "Run 'microk8s inspect' for detailed diagnostics."
fi
echo "======================================"

exit $errors
EOF

chmod +x ~/verify-microk8s.sh

# Exécuter le script
~/verify-microk8s.sh
```

## Vérification de la sécurité

### Permissions et accès

```bash
# Vérifier les permissions des fichiers de configuration
ls -la /var/snap/microk8s/current/credentials/

# Vérifier les certificats
microk8s kubectl config view --raw | grep certificate-authority-data

# Vérifier l'accès RBAC
microk8s kubectl auth can-i --list

# Vérifier les secrets
microk8s kubectl get secrets -A
```

### Ports et services exposés

```bash
# Vérifier les ports en écoute
sudo netstat -tulpn | grep -E ':(16443|10250|10255|10257|10259)'

# Vérifier depuis l'extérieur (si applicable)
# Remplacez <IP> par l'IP de votre machine
curl -k https://<IP>:16443/healthz
```

## Monitoring continu

### Configuration d'alertes simples

```bash
# Script de monitoring basique
cat << 'EOF' > ~/monitor-microk8s.sh
#!/bin/bash

while true; do
    if ! microk8s status | grep -q "is running"; then
        echo "ALERT: MicroK8s is not running at $(date)"
        # Ajouter ici notification (email, slack, etc.)
    fi

    if [ $(microk8s kubectl get nodes | grep -c NotReady) -gt 0 ]; then
        echo "ALERT: Node is NotReady at $(date)"
    fi

    sleep 60
done
EOF

chmod +x ~/monitor-microk8s.sh
```

## Documentation des résultats

Il est recommandé de documenter votre configuration :

```bash
# Créer un rapport de configuration
cat << EOF > ~/microk8s-installation-report.txt
MicroK8s Installation Report
Generated: $(date)
=============================

System Information:
$(uname -a)

MicroK8s Version:
$(microk8s version)

Kubernetes Version:
$(microk8s kubectl version --short)

Node Status:
$(microk8s kubectl get nodes)

Enabled Addons:
$(microk8s status | grep enabled -A 20)

Resource Usage:
$(microk8s kubectl top nodes 2>/dev/null || echo "Metrics not available")

System Pods:
$(microk8s kubectl get pods -A)

Storage:
$(df -h /var/snap/microk8s/)

Network Configuration:
$(ip addr show | grep -E "inet |UP")

EOF

echo "Report saved to ~/microk8s-installation-report.txt"
```

## Points de validation finale

Avant de passer à la configuration avancée, confirmez que :

- ✅ `microk8s status` indique "is running"
- ✅ Le node est en état "Ready"
- ✅ Tous les pods système sont "Running"
- ✅ Vous pouvez créer et supprimer des ressources
- ✅ Le DNS interne fonctionne
- ✅ Les logs ne montrent pas d'erreurs critiques
- ✅ Les ressources système sont suffisantes
- ✅ La connectivité réseau est opérationnelle

## Prochaines étapes

Une fois toutes les vérifications réussies, vous êtes prêt à :

1. Configurer kubectl avec des alias pratiques (section 2.6)
2. Activer les addons nécessaires pour votre lab
3. Configurer le réseau et les domaines
4. Commencer à déployer vos applications

Si certaines vérifications échouent, consultez la section de dépannage spécifique à votre OS ou utilisez `microk8s inspect` pour un diagnostic détaillé.

⏭️
