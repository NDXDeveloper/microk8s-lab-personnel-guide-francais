🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe F - Troubleshooting rapide

## Introduction au Dépannage Kubernetes

Le troubleshooting (dépannage) est une compétence essentielle pour gérer un cluster Kubernetes. Cette annexe vous guide à travers les problèmes les plus courants et leurs solutions, organisés pour un diagnostic rapide et efficace.

### Philosophie du Dépannage

Avant de plonger dans les solutions, adoptez cette approche méthodique :

1. **Observer** : Quel est le symptôme exact ?
2. **Collecter** : Rassembler les logs, événements et descriptions
3. **Analyser** : Identifier la cause racine
4. **Agir** : Appliquer la solution appropriée
5. **Vérifier** : Confirmer que le problème est résolu

### Les Outils de Base du Dépannage

```bash
# Les 5 commandes essentielles pour diagnostiquer
kubectl get pods -o wide              # Vue d'ensemble avec détails
kubectl describe pod POD_NAME         # Informations complètes et événements
kubectl logs POD_NAME                 # Logs de l'application
kubectl get events --sort-by='.lastTimestamp'  # Événements récents
kubectl top pods                      # Utilisation des ressources
```

## Guide de Diagnostic Rapide

### 🔍 Arbre de Décision pour le Diagnostic

```
Le pod fonctionne-t-il ?
├─ NON → Vérifier le statut du pod
│   ├─ Pending → Problème de scheduling (Section 1)
│   ├─ CrashLoopBackOff → Application crash (Section 2)
│   ├─ ImagePullBackOff → Problème d'image (Section 3)
│   ├─ Error → Erreur de configuration (Section 4)
│   └─ Terminating → Pod bloqué (Section 5)
│
└─ OUI → L'application fonctionne-t-elle ?
    ├─ NON → Problèmes applicatifs (Section 6)
    └─ OUI → Est-elle accessible ?
        ├─ NON → Problèmes réseau (Section 7)
        └─ OUI → Performance OK ?
            ├─ NON → Problèmes de ressources (Section 8)
            └─ OUI → Tout va bien ! ✓
```

## Section 1 : Pods en État Pending

### Symptômes
- Le pod reste indéfiniment en état `Pending`
- Le pod n'est jamais programmé sur un node

### Diagnostic Rapide

```bash
# 1. Vérifier l'état du pod
kubectl get pod POD_NAME -o wide

# 2. Voir les détails et événements
kubectl describe pod POD_NAME

# 3. Chercher dans les événements
kubectl get events --field-selector involvedObject.name=POD_NAME
```

### Causes et Solutions

#### Cause 1 : Ressources Insuffisantes

**Symptôme dans les events** :
```
Warning  FailedScheduling  Insufficient cpu
Warning  FailedScheduling  Insufficient memory
```

**Solution** :
```bash
# Vérifier les ressources disponibles
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# Option 1 : Réduire les demandes de ressources
kubectl edit deployment DEPLOYMENT_NAME
# Modifier :
#   resources:
#     requests:
#       memory: "64Mi"  # Réduire
#       cpu: "100m"     # Réduire

# Option 2 : Libérer des ressources
kubectl get pods --all-namespaces | grep -v Running
kubectl delete pod POD_TO_DELETE

# Option 3 : Scaler down d'autres applications
kubectl scale deployment OTHER_APP --replicas=1
```

#### Cause 2 : PVC Non Disponible

**Symptôme dans les events** :
```
Warning  FailedScheduling  pod has unbound immediate PersistentVolumeClaims
```

**Solution** :
```bash
# Vérifier les PVC
kubectl get pvc
kubectl describe pvc PVC_NAME

# Si PVC en Pending, vérifier la StorageClass
kubectl get storageclass

# Pour MicroK8s, activer le storage
microk8s enable hostpath-storage

# Recréer le PVC si nécessaire
kubectl delete pvc PVC_NAME
kubectl apply -f pvc.yaml
```

#### Cause 3 : Node Selector ou Affinity

**Symptôme dans les events** :
```
Warning  FailedScheduling  0/1 nodes are available: 1 node(s) didn't match node selector
```

**Solution** :
```bash
# Vérifier les labels des nodes
kubectl get nodes --show-labels

# Voir le nodeSelector du pod
kubectl get pod POD_NAME -o yaml | grep -A 5 nodeSelector

# Option 1 : Ajouter le label au node
kubectl label node NODE_NAME key=value

# Option 2 : Retirer le nodeSelector
kubectl edit deployment DEPLOYMENT_NAME
# Supprimer la section nodeSelector
```

## Section 2 : CrashLoopBackOff

### Symptômes
- Le pod redémarre constamment
- Statut alterne entre `Running`, `Error` et `CrashLoopBackOff`

### Diagnostic Rapide

```bash
# 1. Voir les redémarrages
kubectl get pod POD_NAME
# Regarder la colonne RESTARTS

# 2. Logs du crash actuel
kubectl logs POD_NAME

# 3. Logs du crash précédent
kubectl logs POD_NAME --previous

# 4. Si plusieurs conteneurs
kubectl logs POD_NAME -c CONTAINER_NAME --previous
```

### Causes et Solutions

#### Cause 1 : Erreur dans l'Application

**Symptôme dans les logs** :
```
Error: Cannot find module '/app/index.js'
Error: Connection refused to database
Segmentation fault
```

**Solution** :
```bash
# Déboguer interactivement
kubectl run -it debug --image=IMAGE_NAME --rm -- /bin/bash

# Vérifier les variables d'environnement
kubectl exec POD_NAME -- env

# Corriger la configuration
kubectl edit deployment DEPLOYMENT_NAME
# Modifier les variables d'environnement ou commandes

# Forcer le redémarrage
kubectl rollout restart deployment/DEPLOYMENT_NAME
```

#### Cause 2 : Problème de Liveness Probe

**Symptôme** : Le pod fonctionne puis est tué après un délai

**Solution** :
```bash
# Voir la configuration de la probe
kubectl describe pod POD_NAME | grep -A 10 "Liveness:"

# Augmenter les délais
kubectl edit deployment DEPLOYMENT_NAME
# Modifier :
#   livenessProbe:
#     initialDelaySeconds: 60  # Plus de temps pour démarrer
#     periodSeconds: 20        # Vérifier moins souvent
#     timeoutSeconds: 10       # Plus de temps pour répondre
#     failureThreshold: 5      # Plus d'essais avant de tuer

# Ou désactiver temporairement pour debug
# Commenter toute la section livenessProbe
```

#### Cause 3 : Dépendances Non Prêtes

**Symptôme** : L'application ne peut pas se connecter à ses dépendances

**Solution** :
```bash
# Ajouter un init container pour attendre
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c', 'until nc -z database-service 5432; do sleep 2; done']
      containers:
      - name: app
        image: APP_IMAGE
EOF
```

## Section 3 : ImagePullBackOff

### Symptômes
- Le pod ne peut pas télécharger l'image Docker
- Statut reste en `ErrImagePull` ou `ImagePullBackOff`

### Diagnostic Rapide

```bash
# 1. Voir l'erreur exacte
kubectl describe pod POD_NAME | grep -A 5 "Events:"

# 2. Vérifier l'image configurée
kubectl get pod POD_NAME -o jsonpath='{.spec.containers[*].image}'
```

### Causes et Solutions

#### Cause 1 : Image Inexistante ou Nom Incorrect

**Symptôme dans les events** :
```
Failed to pull image "nginx:latestt": not found
```

**Solution** :
```bash
# Vérifier le nom exact de l'image
# Docker Hub : https://hub.docker.com/_/nginx
# Ou localement :
docker images

# Corriger le nom de l'image
kubectl edit deployment DEPLOYMENT_NAME
# Corriger : image: nginx:latest  (pas 'latestt')

# Forcer le re-pull
kubectl rollout restart deployment/DEPLOYMENT_NAME
```

#### Cause 2 : Registry Privé Sans Authentification

**Symptôme dans les events** :
```
Failed to pull image "private-registry.com/app:v1": unauthorized
```

**Solution** :
```bash
# Créer un secret pour le registry
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.com \
  --docker-username=USERNAME \
  --docker-password=PASSWORD \
  --docker-email=EMAIL

# Ajouter le secret au deployment
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'

# Ou éditer le deployment
kubectl edit deployment DEPLOYMENT_NAME
# Ajouter :
#   spec:
#     imagePullSecrets:
#     - name: regcred
```

#### Cause 3 : Problème Réseau ou Registry Down

**Solution** :
```bash
# Tester la connectivité
kubectl run test-curl --image=curlimages/curl --rm -it -- sh
curl -I https://registry-1.docker.io

# Pour registry local MicroK8s
microk8s enable registry

# Utiliser le registry local (localhost:32000)
docker tag mon-image:latest localhost:32000/mon-image:latest
docker push localhost:32000/mon-image:latest

# Utiliser dans le deployment
kubectl set image deployment/DEPLOYMENT_NAME app=localhost:32000/mon-image:latest
```

## Section 4 : Erreurs de Configuration

### Symptômes
- Pod en état `Error` ou `CrashLoopBackOff`
- Erreurs liées aux ConfigMaps, Secrets ou Volumes

### Diagnostic Rapide

```bash
# 1. Vérifier les événements
kubectl describe pod POD_NAME

# 2. Vérifier les références manquantes
kubectl get configmap
kubectl get secret
kubectl get pvc
```

### Causes et Solutions

#### Cause 1 : ConfigMap ou Secret Manquant

**Symptôme dans les events** :
```
Warning  Failed  Error: configmap "app-config" not found
Warning  Failed  Error: secret "app-secret" not found
```

**Solution** :
```bash
# Lister les ConfigMaps et Secrets existants
kubectl get configmap
kubectl get secret

# Créer le ConfigMap manquant
kubectl create configmap app-config --from-literal=key=value

# Créer le Secret manquant
kubectl create secret generic app-secret --from-literal=password=secret

# Ou créer depuis un fichier
kubectl create configmap app-config --from-file=./config.yaml
kubectl create secret generic app-secret --from-file=./secret.txt
```

#### Cause 2 : Montage de Volume Incorrect

**Symptôme** :
```
Error: failed to create subPath directory
```

**Solution** :
```bash
# Vérifier la configuration du volume
kubectl get pod POD_NAME -o yaml | grep -A 10 "volumeMounts:"

# Corriger les permissions
kubectl edit deployment DEPLOYMENT_NAME
# Ajouter :
#   securityContext:
#     fsGroup: 2000
#     runAsUser: 1000
#     runAsNonRoot: true
```

#### Cause 3 : Permissions Insuffisantes

**Symptôme** :
```
Error: Permission denied
```

**Solution** :
```bash
# Option 1 : Exécuter en root (déconseillé en prod)
kubectl edit deployment DEPLOYMENT_NAME
# Ajouter :
#   securityContext:
#     runAsUser: 0

# Option 2 : Corriger les permissions du volume
kubectl exec POD_NAME -- chmod -R 755 /path/to/volume
kubectl exec POD_NAME -- chown -R 1000:1000 /path/to/volume
```

## Section 5 : Pods Bloqués en Terminating

### Symptômes
- Le pod reste en état `Terminating` indéfiniment
- `kubectl delete pod` ne fonctionne pas

### Solutions Rapides

```bash
# 1. Forcer la suppression (utiliser avec précaution)
kubectl delete pod POD_NAME --grace-period=0 --force

# 2. Si toujours bloqué, patch le finalizer
kubectl patch pod POD_NAME -p '{"metadata":{"finalizers":null}}'

# 3. Pour plusieurs pods bloqués
kubectl get pods | grep Terminating | awk '{print $1}' | xargs kubectl delete pod --grace-period=0 --force

# 4. Vérifier et nettoyer les finalizers
kubectl get pod POD_NAME -o yaml | grep finalizers -A 5
kubectl patch pod POD_NAME -p '{"metadata":{"finalizers":[]}}'
```

## Section 6 : Problèmes Applicatifs

### Symptômes
- Le pod est `Running` mais l'application ne fonctionne pas
- Erreurs 500, timeouts, comportements inattendus

### Diagnostic et Solutions

#### Diagnostic des Logs

```bash
# Logs en temps réel
kubectl logs -f deployment/DEPLOYMENT_NAME

# Logs avec timestamp
kubectl logs POD_NAME --timestamps

# Logs des dernières minutes
kubectl logs POD_NAME --since=5m

# Pour debug approfondi
kubectl exec -it POD_NAME -- /bin/bash
# Puis dans le conteneur :
ps aux
netstat -tlpn
df -h
cat /etc/hosts
env | sort
```

#### Problèmes de Configuration

```bash
# Vérifier les variables d'environnement
kubectl exec POD_NAME -- env | grep -i database
kubectl exec POD_NAME -- env | grep -i api

# Vérifier les fichiers de config
kubectl exec POD_NAME -- cat /app/config/app.conf

# Tester la connectivité interne
kubectl exec POD_NAME -- nc -zv database-service 5432
kubectl exec POD_NAME -- curl http://api-service/health
```

#### Problèmes de Mémoire

```bash
# Vérifier l'utilisation mémoire
kubectl top pod POD_NAME

# Si OOMKilled (Out Of Memory)
kubectl describe pod POD_NAME | grep -i "OOMKilled"

# Augmenter les limites
kubectl edit deployment DEPLOYMENT_NAME
# Modifier :
#   resources:
#     limits:
#       memory: "512Mi"  # Augmenter
```

## Section 7 : Problèmes Réseau

### Symptômes
- Service inaccessible
- Pas de connectivité entre pods
- Ingress ne fonctionne pas

### Diagnostic et Solutions

#### Test de Connectivité Basique

```bash
# Créer un pod de test
kubectl run test-net --image=nicolaka/netshoot --rm -it -- /bin/bash

# Dans le pod de test :
# Test DNS
nslookup kubernetes.default
nslookup mon-service.mon-namespace.svc.cluster.local

# Test connectivité
ping 8.8.8.8
curl http://mon-service
telnet mon-service 80
```

#### Service Non Accessible

```bash
# Vérifier que le service existe
kubectl get svc SERVICE_NAME

# Vérifier les endpoints
kubectl get endpoints SERVICE_NAME
# Si vide, le selector ne matche aucun pod

# Vérifier les labels
kubectl get pods --show-labels
kubectl get svc SERVICE_NAME -o yaml | grep -A 5 selector

# Corriger les labels si nécessaire
kubectl label pod POD_NAME app=mon-app
```

#### Ingress Non Fonctionnel

```bash
# Vérifier l'ingress
kubectl get ingress
kubectl describe ingress INGRESS_NAME

# Vérifier que l'ingress controller fonctionne
kubectl get pods -n ingress

# Pour MicroK8s, activer ingress
microk8s enable ingress

# Vérifier les annotations
kubectl get ingress INGRESS_NAME -o yaml | grep -A 10 annotations

# Tester localement
kubectl port-forward -n ingress service/nginx-ingress-controller 8080:80
curl -H "Host: mon-domaine.com" http://localhost:8080
```

#### DNS Ne Fonctionne Pas

```bash
# Vérifier CoreDNS
kubectl get pods -n kube-system | grep dns

# Redémarrer CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Tester la résolution DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Si échec, vérifier la config
kubectl get configmap coredns -n kube-system -o yaml
```

## Section 8 : Problèmes de Performance

### Symptômes
- Application lente
- Timeouts fréquents
- CPU ou mémoire saturés

### Diagnostic et Solutions

#### Analyse des Ressources

```bash
# Vue globale
kubectl top nodes
kubectl top pods --all-namespaces

# Pods les plus gourmands
kubectl top pods --all-namespaces | sort -k3 -rn | head -10  # CPU
kubectl top pods --all-namespaces | sort -k4 -rn | head -10  # Mémoire

# Détails d'un pod
kubectl describe pod POD_NAME | grep -A 10 "Limits:"
kubectl describe pod POD_NAME | grep -A 10 "Requests:"
```

#### Optimisation CPU

```bash
# Si throttling CPU
kubectl exec POD_NAME -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled

# Augmenter les limites CPU
kubectl edit deployment DEPLOYMENT_NAME
# Modifier :
#   resources:
#     requests:
#       cpu: "500m"    # Augmenter
#     limits:
#       cpu: "2000m"   # Augmenter

# Ou implémenter HPA
kubectl autoscale deployment DEPLOYMENT_NAME --cpu-percent=70 --min=2 --max=10
```

#### Optimisation Mémoire

```bash
# Vérifier les fuites mémoire
kubectl exec POD_NAME -- cat /proc/meminfo | head -10

# Pour Java, ajuster JVM
kubectl set env deployment/DEPLOYMENT_NAME JAVA_OPTS="-Xmx512m -Xms256m"

# Pour Node.js
kubectl set env deployment/DEPLOYMENT_NAME NODE_OPTIONS="--max-old-space-size=512"
```

## Section 9 : Problèmes de Stockage

### Symptômes
- PVC en `Pending`
- Erreurs d'écriture
- Espace disque plein

### Diagnostic et Solutions

#### PVC Pending

```bash
# Vérifier le PVC
kubectl describe pvc PVC_NAME

# Vérifier la StorageClass
kubectl get storageclass

# Pour MicroK8s
microk8s enable hostpath-storage

# Créer manuellement un PV si nécessaire
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
  storageClassName: manual
EOF
```

#### Espace Disque Plein

```bash
# Vérifier l'espace dans le pod
kubectl exec POD_NAME -- df -h

# Nettoyer les logs
kubectl exec POD_NAME -- find /var/log -type f -name "*.log" -delete

# Nettoyer les fichiers temporaires
kubectl exec POD_NAME -- rm -rf /tmp/*

# Pour le node hôte
df -h /var/lib/microk8s
sudo du -sh /var/lib/microk8s/* | sort -rh | head -10
```

## Section 10 : Problèmes de Certificats

### Symptômes
- Erreurs SSL/TLS
- Certificats expirés
- HTTPS ne fonctionne pas

### Diagnostic et Solutions

```bash
# Vérifier les certificats
kubectl get certificate --all-namespaces
kubectl describe certificate CERT_NAME

# Vérifier cert-manager
kubectl get pods -n cert-manager

# Renouveler un certificat
kubectl delete certificate CERT_NAME
kubectl apply -f certificate.yaml

# Vérifier le secret TLS
kubectl get secret CERT_SECRET -o yaml | grep tls.crt | cut -d' ' -f4 | base64 -d | openssl x509 -text -noout

# Debug Let's Encrypt
kubectl logs -n cert-manager deployment/cert-manager
kubectl describe order
kubectl describe challenge
```

## Check-lists de Dépannage

### 🚀 Check-list de Démarrage Rapide

```bash
#!/bin/bash
# Script de diagnostic initial

echo "=== État du Cluster ==="
kubectl get nodes

echo -e "\n=== Pods en Erreur ==="
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

echo -e "\n=== Événements Récents ==="
kubectl get events --sort-by='.lastTimestamp' | tail -10

echo -e "\n=== Utilisation des Ressources ==="
kubectl top nodes
kubectl top pods --all-namespaces | head -10

echo -e "\n=== Services ==="
kubectl get svc --all-namespaces

echo -e "\n=== Ingress ==="
kubectl get ingress --all-namespaces
```

### 📋 Check-list Pod Non Fonctionnel

1. **État du pod** : `kubectl get pod POD_NAME`
2. **Description** : `kubectl describe pod POD_NAME`
3. **Logs actuels** : `kubectl logs POD_NAME`
4. **Logs précédents** : `kubectl logs POD_NAME --previous`
5. **Événements** : `kubectl get events --field-selector involvedObject.name=POD_NAME`
6. **Ressources** : `kubectl top pod POD_NAME`
7. **Config YAML** : `kubectl get pod POD_NAME -o yaml`

### 🌐 Check-list Problème Réseau

1. **Services** : `kubectl get svc`
2. **Endpoints** : `kubectl get endpoints`
3. **Ingress** : `kubectl get ingress`
4. **DNS Test** : `kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes`
5. **Port-forward test** : `kubectl port-forward service/SERVICE_NAME 8080:80`
6. **NetworkPolicies** : `kubectl get networkpolicies`

### 💾 Check-list Problème de Stockage

1. **PVC Status** : `kubectl get pvc`
2. **PV Status** : `kubectl get pv`
3. **StorageClass** : `kubectl get storageclass`
4. **Espace disque pod** : `kubectl exec POD_NAME -- df -h`
5. **Espace disque node** : `df -h /var/lib/microk8s`

## Scripts de Dépannage Automatisé

### Script de Diagnostic Complet

```bash
#!/bin/bash
# diagnostic-complet.sh
# Diagnostic complet d'un pod problématique

POD_NAME=${1:-}
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
    echo "Usage: ./diagnostic-complet.sh POD_NAME [NAMESPACE]"
    exit 1
fi

echo "========================================"
echo "Diagnostic du pod: $POD_NAME"
echo "Namespace: $NAMESPACE"
echo "Date: $(date)"
echo "========================================"

echo -e "\n[1] État du Pod"
kubectl get pod $POD_NAME -n $NAMESPACE -o wide

echo -e "\n[2] Description Complète"
kubectl describe pod $POD_NAME -n $NAMESPACE | head -50

echo -e "\n[3] Événements Récents"
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME --sort-by='.lastTimestamp'

echo -e "\n[4] Logs Actuels (dernières 20 lignes)"
kubectl logs $POD_NAME -n $NAMESPACE --tail=20 2>/dev/null || echo "Pas de logs disponibles"

echo -e "\n[5] Logs Précédents (si crash)"
kubectl logs $POD_NAME -n $NAMESPACE --previous --tail=20 2>/dev/null || echo "Pas de logs précédents"

echo -e "\n[6] Utilisation des Ressources"
kubectl top pod $POD_NAME -n $NAMESPACE 2>/dev/null || echo "Metrics non disponibles"

echo -e "\n[7] Variables d'Environnement"
kubectl exec $POD_NAME -n $NAMESPACE -- env 2>/dev/null | head -10 || echo "Impossible d'accéder au conteneur"

echo -e "\n[8] Configuration YAML (extraits)"
kubectl get pod $POD_NAME -n $NAMESPACE -o yaml | grep -E "^  (image|state|reason|message):" | head -20

echo -e "\n========================================"
echo "Diagnostic terminé"
```

### Script de Nettoyage d'Urgence

```bash
#!/bin/bash
# cleanup-urgence.sh
# Nettoie les ressources problématiques pour libérer de l'espace

echo "🧹 Nettoyage d'urgence du cluster"

echo -e "\n[1] Suppression des pods Evicted"
kubectl get pods --all-namespaces | grep Evicted | awk '{print $2 " -n " $1}' | xargs -I {} kubectl delete pod {}

echo -e "\n[2] Suppression des pods Completed"
kubectl delete pods --all-namespaces --field-selector=status.phase=Succeeded

echo -e "\n[3] Suppression des pods Failed"
kubectl delete pods --all-namespaces --field-selector=status.phase=Failed

echo -e "\n[4] Suppression des pods Terminating (force)"
kubectl get pods --all-namespaces | grep Terminating | awk '{print $2 " -n " $1}' | xargs -I {} kubectl delete pod {} --grace-period=0 --force

echo -e "\n[5] Nettoyage des images Docker non utilisées"
# Pour MicroK8s
microk8s ctr images prune

echo -e "\n[6] Espace récupéré"
df -h /var/snap/microk8s

echo "✅ Nettoyage terminé"
```

### Script de Monitoring Continu

```bash
#!/bin/bash
# monitor-live.sh
# Monitoring en temps réel du cluster

while true; do
    clear
    echo "========================================="
    echo "Monitoring MicroK8s - $(date '+%Y-%m-%d %H:%M:%S')"
    echo "========================================="

    echo -e "\n📊 NODES"
    kubectl get nodes

    echo -e "\n🔴 PODS EN ERREUR"
    kubectl get pods --all-namespaces | grep -v Running | grep -v Completed | head -5

    echo -e "\n💾 TOP MÉMOIRE"
    kubectl top pods --all-namespaces | sort -k4 -rn | head -5

    echo -e "\n⚡ TOP CPU"
    kubectl top pods --all-namespaces | sort -k3 -rn | head -5

    echo -e "\n📅 ÉVÉNEMENTS RÉCENTS"
    kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -5

    echo -e "\n[Actualisation dans 5 secondes - Ctrl+C pour quitter]"
    sleep 5
done
```

## Référence Rapide des Erreurs

### Tableau des Erreurs Courantes

| Statut | Signification | Action Immédiate |
|--------|--------------|------------------|
| **Pending** | Pod non programmé | Vérifier ressources et PVC |
| **ContainerCreating** | Création en cours | Attendre ou vérifier events |
| **ImagePullBackOff** | Image introuvable | Vérifier nom image et registry |
| **CrashLoopBackOff** | App crash en boucle | Consulter logs --previous |
| **Running** | Pod actif | Vérifier logs si problème app |
| **Terminating** | Suppression en cours | Force delete si bloqué |
| **Evicted** | Évincé (ressources) | Supprimer et augmenter limites |
| **OOMKilled** | Tué (mémoire) | Augmenter limite mémoire |
| **Error** | Erreur générique | Describe pod pour détails |
| **Completed** | Job terminé | Normal pour Jobs |
| **Unknown** | État inconnu | Vérifier node et kubelet |

### Messages d'Erreur Fréquents et Solutions

#### "couldn't find key X in ConfigMap"
```bash
# Vérifier que la clé existe
kubectl get configmap CONFIG_NAME -o yaml
# Ajouter la clé manquante
kubectl edit configmap CONFIG_NAME
```

#### "no space left on device"
```bash
# Nettoyer l'espace disque
df -h
docker system prune -a
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces
```

#### "context deadline exceeded"
```bash
# Timeout réseau, augmenter les timeouts
kubectl edit deployment DEPLOYMENT_NAME
# Augmenter timeoutSeconds dans les probes
```

#### "permission denied"
```bash
# Problème de permissions
kubectl edit deployment DEPLOYMENT_NAME
# Ajouter securityContext approprié
```

## Bonnes Pratiques de Dépannage

### 1. Documentation

- **Gardez un journal** des problèmes rencontrés et solutions
- **Documentez les changements** de configuration
- **Créez des runbooks** pour les incidents récurrents

### 2. Prévention

- **Configurez des alertes** pour détecter les problèmes tôt
- **Utilisez des health checks** (liveness/readiness probes)
- **Définissez des limites de ressources** appropriées
- **Testez en environnement de staging** avant la production

### 3. Outils Recommandés

- **k9s** : Interface TUI pour Kubernetes
- **stern** : Logs multi-pods en temps réel
- **kubectl-debug** : Debug avancé des pods
- **lens** : IDE Kubernetes graphique

### 4. Méthodologie

1. **Commencez simple** : Vérifiez d'abord les bases (pods, logs, events)
2. **Isolez le problème** : Est-ce l'app, le réseau, ou l'infra ?
3. **Reproduisez** : Tentez de reproduire dans un environnement de test
4. **Documentez** : Notez les étapes de résolution pour référence future
5. **Post-mortem** : Analysez la cause racine après résolution

## Cas d'Usage Réels et Solutions

### Cas 1 : Site Web Inaccessible

**Symptômes** : Le site ne répond pas malgré pods Running

**Diagnostic systématique** :
```bash
# 1. Vérifier le pod
kubectl get pod -l app=website
kubectl logs deployment/website

# 2. Vérifier le service
kubectl get svc website-service
kubectl get endpoints website-service

# 3. Vérifier l'ingress
kubectl get ingress website-ingress
kubectl describe ingress website-ingress

# 4. Test local
kubectl port-forward service/website-service 8080:80
curl localhost:8080

# 5. Si OK en local, problème d'Ingress ou DNS
nslookup website.example.com
curl -H "Host: website.example.com" INGRESS_IP
```

### Cas 2 : Base de Données Ne Démarre Pas

**Symptômes** : PostgreSQL en CrashLoopBackOff

**Diagnostic et résolution** :
```bash
# 1. Vérifier les logs
kubectl logs statefulset/postgres --previous

# 2. Souvent problème de permissions sur volume
kubectl exec postgres-0 -- ls -la /var/lib/postgresql/data

# 3. Solution : init container pour permissions
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  template:
    spec:
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ['sh', '-c', 'chmod -R 700 /var/lib/postgresql/data && chown -R 999:999 /var/lib/postgresql/data']
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
EOF
```

### Cas 3 : Déploiement Bloqué à 0/1 Ready

**Symptômes** : Deployment reste à 0/1 READY

**Diagnostic** :
```bash
# 1. Vérifier le rollout
kubectl rollout status deployment/mon-app

# 2. Voir l'historique
kubectl rollout history deployment/mon-app

# 3. Vérifier les replica sets
kubectl get rs -l app=mon-app

# 4. Si bloqué, rollback
kubectl rollout undo deployment/mon-app

# 5. Ou forcer le redémarrage
kubectl rollout restart deployment/mon-app
```

### Cas 4 : Fuite Mémoire Application

**Symptômes** : Pod redémarre périodiquement avec OOMKilled

**Solution** :
```bash
# 1. Confirmer OOMKilled
kubectl describe pod POD_NAME | grep -i "OOMKilled"

# 2. Voir utilisation actuelle
kubectl top pod POD_NAME

# 3. Solution temporaire : augmenter limites
kubectl set resources deployment/APP_NAME --limits=memory=1Gi

# 4. Solution long terme : profiler l'application
# Pour Java
kubectl exec POD_NAME -- jmap -heap PID
# Pour Node.js
kubectl exec POD_NAME -- node --inspect=0.0.0.0:9229

# 5. Configurer restart automatique avec délai
kubectl patch deployment APP_NAME -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","restartPolicy":"Always"}]}}}}'
```

## Commandes de Dépannage par Composant

### MicroK8s Spécifique

```bash
# Status général
microk8s status

# Inspecter les services
microk8s inspect

# Logs des composants
journalctl -u snap.microk8s.daemon-kubelite -f

# Redémarrer MicroK8s
microk8s stop
microk8s start

# Reset complet (ATTENTION: perte de données)
microk8s reset
```

### CoreDNS

```bash
# Vérifier CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Logs CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test résolution DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Redémarrer CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Vérifier la ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

### Ingress Controller

```bash
# Status Ingress Controller
kubectl get pods -n ingress

# Logs Ingress Controller
kubectl logs -n ingress deployment/nginx-ingress-microk8s-controller

# Configuration Nginx
kubectl exec -n ingress deployment/nginx-ingress-microk8s-controller -- cat /etc/nginx/nginx.conf

# Recharger configuration
kubectl exec -n ingress deployment/nginx-ingress-microk8s-controller -- nginx -s reload
```

### Registry Local

```bash
# Vérifier le registry
kubectl get pods -n container-registry

# Tester le registry
curl http://localhost:32000/v2/_catalog

# Push une image
docker tag mon-image:latest localhost:32000/mon-image:latest
docker push localhost:32000/mon-image:latest

# Vérifier les images
curl http://localhost:32000/v2/_catalog
curl http://localhost:32000/v2/mon-image/tags/list
```

## Matrices de Décision

### Matrice de Décision pour Pods Non-Running

```
État du Pod → Action Recommandée
────────────────────────────────
Pending + No nodes available
→ Augmenter nodes ou réduire requests

Pending + Insufficient resources
→ Réduire CPU/memory requests ou libérer ressources

ImagePullBackOff + 404 Not Found
→ Vérifier nom image et tag

ImagePullBackOff + Unauthorized
→ Configurer imagePullSecrets

CrashLoopBackOff + Exit Code 1
→ Vérifier logs application

CrashLoopBackOff + Exit Code 137
→ OOMKilled - augmenter mémoire

Error + Volume mount failed
→ Vérifier PVC et permissions

Terminating > 30s
→ Force delete avec --grace-period=0
```

### Matrice de Performance

```
Symptôme → Métrique → Action
─────────────────────────────
Lenteur → CPU > 80%
→ Scale horizontal ou vertical

Lenteur → Memory > 90%
→ Augmenter limites ou optimiser app

Timeouts → Network latency
→ Vérifier NetworkPolicies et DNS

Erreurs 503 → Pods not ready
→ Ajuster readinessProbe

Throttling → CPU limits atteints
→ Augmenter limits ou optimiser code
```

## Outils de Diagnostic Avancés

### Installation d'Outils Utiles

```bash
# k9s - TUI pour Kubernetes
sudo snap install k9s

# stern - Logs multi-pods
wget https://github.com/stern/stern/releases/latest/download/stern_linux_amd64
sudo mv stern_linux_amd64 /usr/local/bin/stern
sudo chmod +x /usr/local/bin/stern

# kubectl-debug - Debug avancé
kubectl krew install debug

# kubectx/kubens - Changement rapide context/namespace
sudo snap install kubectx --classic
```

### Utilisation des Outils

```bash
# k9s - Interface interactive
k9s

# stern - Logs de plusieurs pods
stern mon-app -n production --tail 50

# kubectl-debug - Debug interactif
kubectl debug POD_NAME -it --image=nicolaka/netshoot

# kubens - Changer de namespace rapidement
kubens production
```

### Création d'un Pod de Debug

```yaml
# debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: default
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```

Utilisation :
```bash
# Déployer le pod de debug
kubectl apply -f debug-pod.yaml

# L'utiliser pour diagnostiquer
kubectl exec -it debug-pod -- /bin/bash

# Dans le pod :
# Test DNS
nslookup kubernetes.default
dig @10.96.0.10 kubernetes.default.svc.cluster.local

# Test réseau
ping 8.8.8.8
traceroute google.com
netstat -tulpn

# Test services
curl http://mon-service.mon-namespace
wget -O- http://mon-service:8080/health

# Capture réseau
tcpdump -i any -w capture.pcap

# Nettoyer après usage
kubectl delete pod debug-pod
```

## Automatisation du Troubleshooting

### Script de Health Check Automatique

```bash
#!/bin/bash
# healthcheck-auto.sh
# Vérifie la santé du cluster et alerte si problème

SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
ERROR_COUNT=0

alert() {
    echo "❌ ALERTE: $1"
    # Envoyer sur Slack (optionnel)
    # curl -X POST -H 'Content-type: application/json' \
    #   --data "{\"text\":\"🚨 MicroK8s Alert: $1\"}" \
    #   $SLACK_WEBHOOK
}

# Check nodes
NODES_NOT_READY=$(kubectl get nodes | grep -v Ready | grep -v NAME | wc -l)
if [ $NODES_NOT_READY -gt 0 ]; then
    alert "$NODES_NOT_READY nodes not ready"
    ((ERROR_COUNT++))
fi

# Check pods système
SYSTEM_PODS_ERROR=$(kubectl get pods -n kube-system | grep -v Running | grep -v Completed | grep -v NAME | wc -l)
if [ $SYSTEM_PODS_ERROR -gt 0 ]; then
    alert "$SYSTEM_PODS_ERROR system pods in error"
    ((ERROR_COUNT++))
fi

# Check pods application
APP_PODS_ERROR=$(kubectl get pods --all-namespaces | grep -E "CrashLoopBackOff|Error|ImagePullBackOff" | wc -l)
if [ $APP_PODS_ERROR -gt 0 ]; then
    alert "$APP_PODS_ERROR application pods in error"
    ((ERROR_COUNT++))
fi

# Check PVC
PVC_PENDING=$(kubectl get pvc --all-namespaces | grep Pending | wc -l)
if [ $PVC_PENDING -gt 0 ]; then
    alert "$PVC_PENDING PVC pending"
    ((ERROR_COUNT++))
fi

# Check certificats (si cert-manager installé)
if kubectl get certificates --all-namespaces &>/dev/null; then
    CERT_NOT_READY=$(kubectl get certificates --all-namespaces | grep False | wc -l)
    if [ $CERT_NOT_READY -gt 0 ]; then
        alert "$CERT_NOT_READY certificates not ready"
        ((ERROR_COUNT++))
    fi
fi

# Résultat
if [ $ERROR_COUNT -eq 0 ]; then
    echo "✅ Cluster healthy - $(date)"
else
    echo "❌ $ERROR_COUNT problems detected - $(date)"
    exit 1
fi
```

### Webhook pour Alertes

```python
#!/usr/bin/env python3
# webhook-alert.py
# Serveur webhook pour recevoir les alertes Kubernetes

from flask import Flask, request, json
import subprocess
import datetime

app = Flask(__name__)

@app.route('/alert', methods=['POST'])
def handle_alert():
    data = request.json

    # Log l'alerte
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open('/var/log/k8s-alerts.log', 'a') as f:
        f.write(f"[{timestamp}] {json.dumps(data)}\n")

    # Action automatique selon le type d'alerte
    if 'OOMKilled' in str(data):
        # Augmenter automatiquement la mémoire
        subprocess.run(['kubectl', 'set', 'resources',
                       'deployment', data['deployment'],
                       '--limits=memory=1Gi'])

    return {'status': 'received'}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Tableaux de Référence Rapide

### Codes de Sortie Courants

| Code | Signification | Cause Probable |
|------|--------------|----------------|
| 0 | Succès | Normal |
| 1 | Erreur générale | Erreur application |
| 125 | Container failed to run | Erreur Docker |
| 126 | Command not executable | Permission denied |
| 127 | Command not found | Binary manquant |
| 128+n | Killed by signal n | Ex: 137 = SIGKILL (OOM) |
| 130 | Killed by Ctrl+C | SIGINT |
| 137 | Out of Memory | OOMKilled |
| 139 | Segmentation fault | Bug application |
| 143 | SIGTERM | Terminated gracefully |

### Signaux Linux dans Kubernetes

| Signal | Numéro | Usage Kubernetes |
|--------|--------|------------------|
| SIGTERM | 15 | Arrêt gracieux (default) |
| SIGKILL | 9 | Force kill après grace period |
| SIGINT | 2 | Interruption (Ctrl+C) |
| SIGHUP | 1 | Reload configuration |
| SIGUSR1 | 10 | Custom (debug, dumps) |
| SIGUSR2 | 12 | Custom (reload, rotate) |

### Ports Standards dans Kubernetes

| Port | Service | Description |
|------|---------|-------------|
| 6443 | API Server | API Kubernetes |
| 2379-2380 | etcd | Base de données |
| 10250 | Kubelet | API Kubelet |
| 10251 | Scheduler | Scheduler metrics |
| 10252 | Controller | Controller metrics |
| 10256 | kube-proxy | Health checks |
| 30000-32767 | NodePort | Services NodePort |
| 53 | CoreDNS | DNS interne |
| 80/443 | Ingress | HTTP/HTTPS |
| 32000 | Registry | Registry local MicroK8s |

## Conclusion

Le troubleshooting efficace dans Kubernetes nécessite :

1. **Une approche méthodique** : Suivre une checklist systématique
2. **Les bons outils** : kubectl, logs, describe, events
3. **La compréhension des patterns** : Reconnaître les erreurs courantes
4. **La documentation** : Noter les solutions pour référence future
5. **L'automatisation** : Scripts pour diagnostics répétitifs

Les problèmes les plus courants ont généralement des solutions simples :
- **Pending** → Vérifier ressources et PVC
- **CrashLoopBackOff** → Consulter les logs
- **ImagePullBackOff** → Vérifier image et credentials
- **Service inaccessible** → Vérifier labels et selectors

Avec cette annexe comme référence, vous devriez pouvoir résoudre rapidement 90% des problèmes rencontrés dans votre lab MicroK8s. Pour les 10% restants, la communauté Kubernetes et les forums sont d'excellentes ressources.

---

*Note : Cette annexe est un document vivant. Enrichissez-la avec vos propres cas rencontrés et solutions trouvées. Chaque problème résolu est une opportunité d'apprentissage et d'amélioration de vos compétences Kubernetes.*

⏭️
