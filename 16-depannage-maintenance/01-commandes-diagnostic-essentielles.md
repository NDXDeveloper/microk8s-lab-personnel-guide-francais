🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1 Commandes de diagnostic essentielles

## Introduction aux commandes de diagnostic

Lorsque votre cluster MicroK8s présente un comportement inattendu, disposer des bonnes commandes de diagnostic fait toute la différence entre des heures de frustration et une résolution rapide du problème. Cette section vous présente les commandes essentielles, organisées par ordre de fréquence d'utilisation et par type de problème à diagnostiquer.

## Vérification de l'état général du cluster

### État de MicroK8s

La première vérification à effectuer concerne l'état général de MicroK8s lui-même. Cette commande simple vous donne un aperçu immédiat de la santé de votre cluster :

```bash
microk8s status
```

Cette commande affiche l'état de MicroK8s (running/not running) et liste tous les addons activés. Si MicroK8s n'est pas en cours d'exécution, vous pouvez le démarrer avec :

```bash
microk8s start
```

Pour une inspection plus approfondie du système, MicroK8s fournit une commande spécialisée qui génère un rapport complet :

```bash
microk8s inspect
```

Cette commande créé un fichier tarball contenant des informations de diagnostic détaillées incluant les logs système, la configuration réseau, l'état des services, et les métriques de performance. C'est particulièrement utile lorsque vous devez partager des informations de diagnostic avec la communauté pour obtenir de l'aide.

### Vérification des nœuds

Pour vérifier l'état du ou des nœuds de votre cluster :

```bash
microk8s kubectl get nodes
```

Cette commande affiche tous les nœuds avec leur statut (Ready/NotReady), leurs rôles, leur version Kubernetes, et depuis combien de temps ils sont en fonction. Pour plus de détails sur un nœud spécifique :

```bash
microk8s kubectl describe node <nom-du-noeud>
```

La commande `describe` révèle des informations cruciales comme les conditions du nœud (mémoire, disque, PID disponibles), les capacités en ressources (CPU, mémoire), les pods en cours d'exécution, et les événements récents concernant le nœud.

### Informations sur le cluster

Pour obtenir des informations générales sur votre cluster Kubernetes :

```bash
microk8s kubectl cluster-info
```

Cette commande affiche les URLs des principaux composants du cluster comme le serveur API Kubernetes et CoreDNS. C'est utile pour vérifier que les services essentiels sont accessibles.

## Diagnostic des pods et des applications

### Liste et état des pods

La commande la plus fréquemment utilisée pour vérifier l'état de vos applications :

```bash
# Tous les pods dans le namespace par défaut
microk8s kubectl get pods

# Tous les pods dans tous les namespaces
microk8s kubectl get pods --all-namespaces

# Pods dans un namespace spécifique
microk8s kubectl get pods -n <namespace>

# Affichage détaillé avec plus d'informations
microk8s kubectl get pods -o wide
```

L'option `-o wide` est particulièrement utile car elle affiche des colonnes supplémentaires comme l'IP du pod, le nœud sur lequel il s'exécute, et les nominations.

### Analyse détaillée d'un pod

Quand un pod présente des problèmes, la commande `describe` devient votre meilleur allié :

```bash
microk8s kubectl describe pod <nom-du-pod> -n <namespace>
```

Cette commande fournit une vue exhaustive incluant :
- Les métadonnées du pod (labels, annotations)
- L'état de chaque conteneur dans le pod
- Les volumes montés
- Les conditions du pod
- Les événements récents (très important pour le diagnostic)

### Consultation des logs

Les logs sont essentiels pour comprendre ce qui se passe à l'intérieur de vos applications :

```bash
# Logs du pod (conteneur par défaut)
microk8s kubectl logs <nom-du-pod> -n <namespace>

# Logs d'un conteneur spécifique dans un pod multi-conteneurs
microk8s kubectl logs <nom-du-pod> -c <nom-du-conteneur> -n <namespace>

# Suivre les logs en temps réel (comme tail -f)
microk8s kubectl logs -f <nom-du-pod> -n <namespace>

# Afficher les logs précédents (si le conteneur a redémarré)
microk8s kubectl logs <nom-du-pod> -n <namespace> --previous

# Limiter le nombre de lignes affichées
microk8s kubectl logs <nom-du-pod> -n <namespace> --tail=100

# Logs avec timestamp
microk8s kubectl logs <nom-du-pod> -n <namespace> --timestamps
```

Pour les déploiements avec plusieurs réplicas, vous pouvez voir les logs de tous les pods d'un déploiement :

```bash
microk8s kubectl logs -l app=<label> -n <namespace>
```

### Exécution de commandes dans les pods

Parfois, vous devez investiguer directement à l'intérieur d'un conteneur :

```bash
# Ouvrir un shell interactif dans un pod
microk8s kubectl exec -it <nom-du-pod> -n <namespace> -- /bin/bash

# Si bash n'est pas disponible, essayer sh
microk8s kubectl exec -it <nom-du-pod> -n <namespace> -- /bin/sh

# Exécuter une commande spécifique
microk8s kubectl exec <nom-du-pod> -n <namespace> -- ls -la /app

# Pour un pod multi-conteneurs, spécifier le conteneur
microk8s kubectl exec -it <nom-du-pod> -c <nom-du-conteneur> -n <namespace> -- /bin/bash
```

## Diagnostic des services et du réseau

### Vérification des services

Les services Kubernetes exposent vos applications. Pour les diagnostiquer :

```bash
# Liste tous les services
microk8s kubectl get services --all-namespaces

# Détails d'un service spécifique
microk8s kubectl describe service <nom-du-service> -n <namespace>

# Vérifier les endpoints d'un service
microk8s kubectl get endpoints <nom-du-service> -n <namespace>
```

Les endpoints montrent les adresses IP réelles des pods derrière un service. Si un service ne fonctionne pas, vérifier que les endpoints sont correctement peuplés est crucial.

### Test de connectivité réseau

Pour tester la connectivité réseau depuis l'intérieur du cluster, vous pouvez utiliser un pod de test temporaire :

```bash
# Créer un pod de test avec des outils réseau
microk8s kubectl run test-pod --image=busybox:latest --rm -it -- /bin/sh

# Depuis ce pod, vous pouvez tester :
# DNS
nslookup kubernetes.default
nslookup <nom-du-service>.<namespace>.svc.cluster.local

# Connectivité
ping <ip-du-pod>
wget -O- http://<nom-du-service>.<namespace>.svc.cluster.local
```

### Diagnostic DNS

Le DNS est souvent source de problèmes. Pour le diagnostiquer :

```bash
# Vérifier que CoreDNS est en cours d'exécution
microk8s kubectl get pods -n kube-system -l k8s-app=kube-dns

# Voir les logs de CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns

# Tester la résolution DNS depuis un pod
microk8s kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

## Diagnostic des ressources et performances

### Utilisation des ressources par les pods

Pour identifier les pods consommant le plus de ressources :

```bash
# Utilisation CPU et mémoire des pods
microk8s kubectl top pods --all-namespaces

# Utilisation par les nœuds
microk8s kubectl top nodes

# Trier les pods par utilisation CPU
microk8s kubectl top pods --all-namespaces --sort-by=cpu

# Trier par mémoire
microk8s kubectl top pods --all-namespaces --sort-by=memory
```

Note : Ces commandes nécessitent que metrics-server soit activé :
```bash
microk8s enable metrics-server
```

### Vérification des limites de ressources

Pour voir les ressources demandées et les limites des pods :

```bash
# Voir les requests et limits d'un pod
microk8s kubectl describe pod <nom-du-pod> -n <namespace> | grep -A 5 "Limits\|Requests"

# Vue d'ensemble des ressources dans un namespace
microk8s kubectl describe resourcequota -n <namespace>
microk8s kubectl describe limitrange -n <namespace>
```

## Diagnostic des volumes et du stockage

### Vérification des Persistent Volumes

Pour diagnostiquer les problèmes de stockage :

```bash
# Liste des Persistent Volumes
microk8s kubectl get pv

# Liste des Persistent Volume Claims
microk8s kubectl get pvc --all-namespaces

# Détails d'un PVC
microk8s kubectl describe pvc <nom-du-pvc> -n <namespace>

# Vérifier les StorageClasses disponibles
microk8s kubectl get storageclass
```

### Diagnostic des problèmes de montage

Si un pod ne peut pas monter un volume :

```bash
# Vérifier les événements du pod
microk8s kubectl describe pod <nom-du-pod> -n <namespace> | grep -A 10 Events

# Vérifier l'état du PVC
microk8s kubectl get pvc <nom-du-pvc> -n <namespace>
```

## Événements et historique

### Consultation des événements

Les événements Kubernetes fournissent un historique des actions dans le cluster :

```bash
# Tous les événements récents
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Événements d'un namespace spécifique
microk8s kubectl get events -n <namespace>

# Événements liés à un objet spécifique
microk8s kubectl get events --field-selector involvedObject.name=<nom-du-pod> -n <namespace>

# Surveiller les événements en temps réel
microk8s kubectl get events --all-namespaces -w
```

## Diagnostic des déploiements et mises à jour

### État des déploiements

Pour diagnostiquer les problèmes de déploiement :

```bash
# État de tous les déploiements
microk8s kubectl get deployments --all-namespaces

# Historique d'un déploiement
microk8s kubectl rollout history deployment/<nom-du-deployment> -n <namespace>

# État du rollout
microk8s kubectl rollout status deployment/<nom-du-deployment> -n <namespace>

# Détails d'un déploiement
microk8s kubectl describe deployment <nom-du-deployment> -n <namespace>
```

### Diagnostic des ReplicaSets

Les ReplicaSets gèrent les réplicas des pods :

```bash
# Liste des ReplicaSets
microk8s kubectl get replicasets -n <namespace>

# Voir pourquoi les pods ne sont pas créés
microk8s kubectl describe replicaset <nom-du-rs> -n <namespace>
```

## Commandes système complémentaires

### Vérification des processus MicroK8s

Au niveau système Linux, vous pouvez vérifier les processus :

```bash
# Voir tous les processus MicroK8s
ps aux | grep microk8s

# Vérifier l'utilisation des ressources système
htop  # ou top si htop n'est pas installé

# Espace disque disponible
df -h

# Utilisation de la mémoire
free -h

# Logs système pour MicroK8s
journalctl -u snap.microk8s.daemon-kubelite -f
```

### Vérification des ports

Pour vérifier que les ports nécessaires sont ouverts :

```bash
# Ports en écoute
sudo netstat -tlnp | grep microk8s
# ou
sudo ss -tlnp | grep microk8s

# Vérifier un port spécifique
sudo lsof -i :16443  # Port API Kubernetes
```

## Création de rapports de diagnostic

### Génération d'un rapport complet

Pour créer un rapport de diagnostic complet à partager :

```bash
# Créer un script de diagnostic
cat > diagnostic.sh << 'EOF'
#!/bin/bash
echo "=== MicroK8s Status ==="
microk8s status

echo -e "\n=== Nodes ==="
microk8s kubectl get nodes -o wide

echo -e "\n=== Pods (all namespaces) ==="
microk8s kubectl get pods --all-namespaces -o wide

echo -e "\n=== Services ==="
microk8s kubectl get services --all-namespaces

echo -e "\n=== Events (last 50) ==="
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -50

echo -e "\n=== System Resources ==="
df -h
free -h

echo -e "\n=== Recent System Logs ==="
journalctl -u snap.microk8s.daemon-kubelite --since "1 hour ago" --no-pager | tail -100
EOF

chmod +x diagnostic.sh
./diagnostic.sh > diagnostic_report_$(date +%Y%m%d_%H%M%S).txt
```

## Bonnes pratiques pour le diagnostic

### Organisation des commandes

Créez des alias pour les commandes fréquentes dans votre `.bashrc` ou `.zshrc` :

```bash
# Alias utiles pour MicroK8s
alias mk='microk8s kubectl'
alias mkgp='microk8s kubectl get pods'
alias mkgpa='microk8s kubectl get pods --all-namespaces'
alias mkgs='microk8s kubectl get services'
alias mkgn='microk8s kubectl get nodes'
alias mkaf='microk8s kubectl apply -f'
alias mkdp='microk8s kubectl describe pod'
alias mkl='microk8s kubectl logs'
alias mklf='microk8s kubectl logs -f'
```

### Documentation des problèmes

Gardez un journal des problèmes rencontrés et leurs solutions :

```bash
# Créer un fichier de log personnel
echo "$(date): Problème: [description] | Solution: [commandes utilisées]" >> ~/microk8s_troubleshooting.log
```

### Approche méthodique

Quand vous diagnostiquez un problème, suivez toujours cette séquence :

1. **Vérifier l'état général** : `microk8s status` et `kubectl get nodes`
2. **Identifier les composants affectés** : `kubectl get pods --all-namespaces`
3. **Examiner les événements** : `kubectl get events --sort-by='.lastTimestamp'`
4. **Consulter les logs** : `kubectl logs` pour les pods problématiques
5. **Vérifier les ressources** : `kubectl top pods` et `df -h`, `free -h`
6. **Analyser en détail** : `kubectl describe` sur les ressources problématiques

## Conclusion

La maîtrise de ces commandes de diagnostic constitue la base de votre capacité à maintenir un cluster MicroK8s sain. Avec la pratique, vous développerez des réflexes qui vous permettront d'identifier rapidement les problèmes. N'hésitez pas à créer vos propres scripts combinant plusieurs de ces commandes pour automatiser les diagnostics récurrents. Dans la section suivante, nous approfondirons l'analyse des logs, un aspect crucial du diagnostic qui mérite une attention particulière.

⏭️
