🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe E - Commandes kubectl essentielles

## Introduction à kubectl

kubectl (prononcé "cube-control" ou "cube-C-T-L") est l'outil en ligne de commande pour interagir avec votre cluster Kubernetes. C'est votre interface principale pour gérer les applications, inspecter les ressources et diagnostiquer les problèmes dans MicroK8s.

### Qu'est-ce que kubectl ?

kubectl est comme une télécommande universelle pour votre cluster Kubernetes. Il permet de :
- **Créer et gérer** des ressources (pods, services, deployments)
- **Inspecter** l'état du cluster et des applications
- **Déboguer** les problèmes et consulter les logs
- **Modifier** les configurations en temps réel
- **Copier** des fichiers vers/depuis les conteneurs

### Configuration pour MicroK8s

Avec MicroK8s, kubectl est intégré. Vous avez deux options :

```bash
# Option 1 : Utiliser directement microk8s kubectl
microk8s kubectl get nodes

# Option 2 : Créer un alias (recommandé)
echo "alias kubectl='microk8s kubectl'" >> ~/.bashrc
echo "alias k='microk8s kubectl'" >> ~/.bashrc
source ~/.bashrc

# Maintenant vous pouvez utiliser :
kubectl get nodes
k get nodes  # Version courte
```

### Structure d'une Commande kubectl

La plupart des commandes kubectl suivent cette structure :

```bash
kubectl [action] [type-ressource] [nom-ressource] [options]

# Exemples :
kubectl get pods                    # Action: get, Type: pods
kubectl delete service mon-service  # Action: delete, Type: service, Nom: mon-service
kubectl logs mon-pod -f             # Action: logs, Nom: mon-pod, Option: -f (follow)
```

## Commandes de Base

### 1. Affichage des Ressources (GET)

```bash
# === Afficher les ressources ===

# Pods (unités de base contenant vos conteneurs)
kubectl get pods                          # Tous les pods du namespace actuel
kubectl get pods -n kube-system          # Pods d'un namespace spécifique
kubectl get pods --all-namespaces        # Tous les pods de tous les namespaces
kubectl get pods -o wide                 # Vue détaillée avec IP et node
kubectl get pod mon-pod -o yaml          # Afficher en YAML
kubectl get pod mon-pod -o json          # Afficher en JSON

# Deployments (gèrent les répliques de pods)
kubectl get deployments                   # Liste des deployments
kubectl get deploy                        # Version courte
kubectl get deployment mon-app -o wide   # Détails d'un deployment

# Services (exposition réseau des pods)
kubectl get services                      # Liste des services
kubectl get svc                           # Version courte
kubectl get svc mon-service -o wide      # Détails d'un service

# Nodes (machines du cluster)
kubectl get nodes                         # Liste des nodes
kubectl get nodes -o wide                # Avec plus d'infos (IP, version, OS)
kubectl get node mon-node -o yaml        # Configuration complète d'un node

# Namespaces (espaces de travail isolés)
kubectl get namespaces                    # Liste des namespaces
kubectl get ns                            # Version courte

# Tout afficher
kubectl get all                          # Toutes les ressources principales
kubectl get all -n mon-namespace         # Dans un namespace spécifique
kubectl get all --all-namespaces         # Partout

# Events (historique des événements)
kubectl get events                       # Événements récents
kubectl get events --sort-by='.lastTimestamp'  # Triés par date
kubectl get events -w                    # Watch en temps réel
```

### 2. Création de Ressources (CREATE/APPLY)

```bash
# === Créer des ressources ===

# À partir d'un fichier YAML
kubectl apply -f mon-app.yaml            # Créer ou mettre à jour
kubectl create -f mon-app.yaml           # Créer seulement (erreur si existe)

# Créer un deployment rapidement
kubectl create deployment nginx --image=nginx:latest
kubectl create deployment mon-app --image=mon-image:1.0 --replicas=3

# Créer un service
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment mon-app --port=8080 --type=NodePort

# Créer un namespace
kubectl create namespace developpement
kubectl create ns prod  # Version courte

# Créer un secret
kubectl create secret generic mon-secret --from-literal=password=secret123
kubectl create secret generic api-keys --from-file=./api-key.txt

# Créer une ConfigMap
kubectl create configmap app-config --from-literal=env=production
kubectl create configmap nginx-config --from-file=./nginx.conf

# Appliquer plusieurs fichiers
kubectl apply -f ./k8s/                  # Tous les YAML d'un dossier
kubectl apply -f app.yaml -f service.yaml  # Plusieurs fichiers
```

### 3. Mise à Jour des Ressources (EDIT/SET/SCALE)

```bash
# === Modifier des ressources ===

# Éditer directement (ouvre un éditeur)
kubectl edit deployment mon-app          # Édition interactive
kubectl edit svc mon-service             # Modifier un service

# Mettre à jour l'image d'un deployment
kubectl set image deployment/mon-app mon-app=mon-image:2.0
kubectl set image deploy/nginx nginx=nginx:1.21

# Changer les ressources
kubectl set resources deployment mon-app -c=mon-conteneur --limits=memory=512Mi,cpu=500m

# Scaler (changer le nombre de répliques)
kubectl scale deployment mon-app --replicas=5
kubectl scale deploy nginx --replicas=0  # Arrêter sans supprimer

# Autoscale
kubectl autoscale deployment mon-app --min=2 --max=10 --cpu-percent=80

# Rollout (déploiement progressif)
kubectl rollout status deployment/mon-app     # Vérifier le statut
kubectl rollout history deployment/mon-app    # Historique des versions
kubectl rollout undo deployment/mon-app       # Annuler le dernier déploiement
kubectl rollout undo deployment/mon-app --to-revision=2  # Revenir à une version
kubectl rollout restart deployment/mon-app    # Redémarrer tous les pods
kubectl rollout pause deployment/mon-app      # Mettre en pause
kubectl rollout resume deployment/mon-app     # Reprendre
```

### 4. Suppression de Ressources (DELETE)

```bash
# === Supprimer des ressources ===

# Supprimer par nom
kubectl delete pod mon-pod
kubectl delete deployment mon-app
kubectl delete service mon-service
kubectl delete namespace test

# Supprimer depuis un fichier
kubectl delete -f mon-app.yaml
kubectl delete -f ./k8s/                 # Tous les fichiers d'un dossier

# Supprimer par label
kubectl delete pods -l app=test          # Tous les pods avec label app=test
kubectl delete all -l environment=dev    # Toutes ressources avec ce label

# Supprimer tout dans un namespace
kubectl delete all --all -n test         # ATTENTION : supprime tout !

# Force delete (pour pods bloqués)
kubectl delete pod mon-pod --grace-period=0 --force

# Supprimer les pods terminés
kubectl delete pods --field-selector=status.phase=Succeeded
kubectl delete pods --field-selector=status.phase=Failed
```

## Commandes de Diagnostic

### 5. Description Détaillée (DESCRIBE)

```bash
# === Obtenir des informations détaillées ===

# Describe affiche les détails, événements et configuration
kubectl describe pod mon-pod
kubectl describe deployment mon-app
kubectl describe service mon-service
kubectl describe node mon-node

# Informations spécifiques
kubectl describe ingress mon-ingress     # Règles de routage
kubectl describe pvc mon-pvc             # État du volume
kubectl describe secret mon-secret       # Sans afficher les valeurs

# Voir les événements d'une ressource
kubectl describe pod mon-pod | grep -A 10 Events:
```

### 6. Logs et Débogage (LOGS/EXEC/CP)

```bash
# === Consulter les logs ===

# Logs d'un pod
kubectl logs mon-pod                     # Logs actuels
kubectl logs mon-pod -f                  # Suivre en temps réel (comme tail -f)
kubectl logs mon-pod --tail=100          # 100 dernières lignes
kubectl logs mon-pod --since=1h          # Logs de la dernière heure
kubectl logs mon-pod --timestamps        # Avec horodatage

# Pod avec plusieurs conteneurs
kubectl logs mon-pod -c mon-conteneur    # Conteneur spécifique
kubectl logs mon-pod --all-containers    # Tous les conteneurs

# Logs précédents (après crash)
kubectl logs mon-pod --previous
kubectl logs mon-pod -p                  # Version courte

# Logs d'un deployment (tous les pods)
kubectl logs deployment/mon-app
kubectl logs deploy/mon-app --tail=50 -f

# Logs par label
kubectl logs -l app=mon-app --tail=10

# === Exécuter des commandes dans un conteneur ===

# Shell interactif
kubectl exec -it mon-pod -- /bin/bash
kubectl exec -it mon-pod -- /bin/sh      # Si bash non disponible

# Commande simple
kubectl exec mon-pod -- ls /app
kubectl exec mon-pod -- cat /etc/hostname

# Avec plusieurs conteneurs
kubectl exec -it mon-pod -c mon-conteneur -- /bin/bash

# === Copier des fichiers ===

# Copier depuis le pod vers local
kubectl cp mon-pod:/app/logs/app.log ./app.log
kubectl cp mon-namespace/mon-pod:/data ./local-data

# Copier depuis local vers le pod
kubectl cp ./config.yaml mon-pod:/app/config.yaml
kubectl cp ./data mon-pod:/app/data      # Dossier entier

# Avec conteneur spécifique
kubectl cp mon-pod:/logs ./logs -c mon-conteneur
```

### 7. Port Forwarding et Proxy

```bash
# === Accès local aux services ===

# Port-forward vers un pod
kubectl port-forward mon-pod 8080:80     # Local:8080 -> Pod:80
kubectl port-forward mon-pod 8080:80 9090:90  # Plusieurs ports

# Port-forward vers un service
kubectl port-forward service/mon-service 8080:80
kubectl port-forward svc/mon-service 3000:3000

# Port-forward vers un deployment
kubectl port-forward deployment/mon-app 8080:8080

# Écouter sur toutes les interfaces (pas seulement localhost)
kubectl port-forward --address 0.0.0.0 mon-pod 8080:80

# Proxy vers l'API Kubernetes
kubectl proxy                             # Démarre sur port 8001
kubectl proxy --port=8080                # Port personnalisé

# Accès au dashboard via proxy
kubectl proxy
# Puis ouvrir : http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## Commandes d'Inspection

### 8. Monitoring des Ressources (TOP)

```bash
# === Surveillance de l'utilisation des ressources ===
# Note : nécessite metrics-server

# Utilisation des nodes
kubectl top nodes                        # CPU et mémoire par node
kubectl top node mon-node               # Un node spécifique

# Utilisation des pods
kubectl top pods                         # Dans le namespace actuel
kubectl top pods --all-namespaces       # Tous les pods
kubectl top pods -n kube-system         # Namespace spécifique
kubectl top pod mon-pod                 # Un pod spécifique

# Trier par utilisation
kubectl top pods --sort-by=cpu          # Tri par CPU
kubectl top pods --sort-by=memory       # Tri par mémoire

# Avec les conteneurs
kubectl top pod mon-pod --containers    # Détail par conteneur
```

### 9. Informations du Cluster (CLUSTER-INFO/VERSION)

```bash
# === Informations sur le cluster ===

# Informations générales
kubectl cluster-info                     # URLs des services principaux
kubectl cluster-info dump                # Dump complet pour diagnostic

# Version
kubectl version                          # Version client et serveur
kubectl version --short                  # Version courte
kubectl version -o json                  # Format JSON

# API resources disponibles
kubectl api-resources                    # Liste toutes les ressources
kubectl api-resources --namespaced=true  # Ressources avec namespace
kubectl api-resources --namespaced=false # Ressources globales
kubectl api-resources -o wide            # Avec verbes supportés

# API versions
kubectl api-versions                     # Versions d'API disponibles

# Configuration
kubectl config view                      # Configuration kubectl
kubectl config current-context           # Contexte actuel
kubectl config get-contexts              # Tous les contextes
```

## Commandes Avancées

### 10. Labels et Sélecteurs

```bash
# === Gestion des labels ===

# Ajouter un label
kubectl label pod mon-pod env=production
kubectl label deployment mon-app version=2.0

# Modifier un label (overwrite)
kubectl label pod mon-pod env=staging --overwrite

# Supprimer un label
kubectl label pod mon-pod env-          # Note le "-" à la fin

# Afficher avec labels
kubectl get pods --show-labels
kubectl get pods -L env,version         # Colonnes spécifiques

# Filtrer par label
kubectl get pods -l env=production
kubectl get pods -l 'env in (dev,staging)'
kubectl get pods -l 'version!=1.0'
kubectl get all -l app=mon-app

# === Annotations ===

# Ajouter une annotation
kubectl annotate pod mon-pod description="Pod de production"
kubectl annotate deployment mon-app contact="admin@example.com"

# Supprimer une annotation
kubectl annotate pod mon-pod description-
```

### 11. Gestion des Namespaces

```bash
# === Travailler avec les namespaces ===

# Créer un namespace
kubectl create namespace developpement
kubectl create ns prod

# Lister les namespaces
kubectl get namespaces
kubectl get ns

# Changer le namespace par défaut
kubectl config set-context --current --namespace=developpement

# Vérifier le namespace actuel
kubectl config view --minify | grep namespace:

# Commandes avec namespace
kubectl get pods -n production          # Dans un namespace
kubectl get pods --all-namespaces       # Tous les namespaces
kubectl get pods -A                     # Version courte pour all-namespaces

# Supprimer un namespace (supprime tout dedans !)
kubectl delete namespace test
```

### 12. Gestion des Contexts

```bash
# === Contexts (connexions à différents clusters) ===

# Voir les contexts
kubectl config get-contexts              # Liste tous les contexts
kubectl config current-context           # Context actuel

# Changer de context
kubectl config use-context mon-cluster

# Créer un nouveau context
kubectl config set-context dev-context \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=development

# Modifier le context actuel
kubectl config set-context --current --namespace=production

# Supprimer un context
kubectl config delete-context old-context

# Voir la config complète
kubectl config view
kubectl config view --minify             # Seulement le context actuel
```

### 13. Dry-Run et Validation

```bash
# === Tester sans appliquer ===

# Dry-run côté client (validation syntaxe)
kubectl apply -f mon-app.yaml --dry-run=client

# Dry-run côté serveur (validation complète)
kubectl apply -f mon-app.yaml --dry-run=server

# Générer YAML sans créer
kubectl create deployment test --image=nginx --dry-run=client -o yaml

# Générer et sauvegarder
kubectl create deployment test --image=nginx --dry-run=client -o yaml > test.yaml

# Valider un fichier
kubectl apply -f mon-app.yaml --validate=true

# Diff avant application
kubectl diff -f mon-app-v2.yaml         # Voir les changements
```

### 14. JSONPath et Output Formatting

```bash
# === Extraction de données spécifiques ===

# Formats de sortie
kubectl get pods -o wide                 # Tableau étendu
kubectl get pods -o yaml                 # YAML complet
kubectl get pods -o json                 # JSON complet
kubectl get pods -o name                 # Juste les noms

# JSONPath pour extraction
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image

# Templating Go
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'

# Exemples pratiques JSONPath
# IP d'un pod
kubectl get pod mon-pod -o jsonpath='{.status.podIP}'

# Image d'un conteneur
kubectl get pod mon-pod -o jsonpath='{.spec.containers[0].image}'

# Tous les noms de services
kubectl get services -o jsonpath='{.items[*].metadata.name}'
```

## Commandes de Maintenance

### 15. Drain et Cordon

```bash
# === Maintenance des nodes ===

# Marquer un node comme non-programmable
kubectl cordon mon-node                  # Empêche nouveaux pods
kubectl uncordon mon-node                # Réactive le node

# Vider un node (déplace les pods)
kubectl drain mon-node                   # Évacue les pods
kubectl drain mon-node --ignore-daemonsets  # Ignore les DaemonSets
kubectl drain mon-node --delete-emptydir-data  # Supprime données locales
kubectl drain mon-node --force           # Force même si pods non gérés

# Taint (marquer un node avec condition)
kubectl taint nodes mon-node key=value:NoSchedule
kubectl taint nodes mon-node key-        # Retirer le taint
```

### 16. Patch et Replace

```bash
# === Modifications avancées ===

# Patch (modification partielle)
kubectl patch deployment mon-app -p '{"spec":{"replicas":3}}'
kubectl patch pod mon-pod -p '{"metadata":{"labels":{"env":"prod"}}}'

# Patch avec merge stratégique
kubectl patch deployment mon-app --type='strategic' -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","image":"app:v2"}]}}}}'

# Replace (remplacement complet)
kubectl get deployment mon-app -o yaml | sed 's/v1/v2/g' | kubectl replace -f -

# Replace avec force
kubectl replace --force -f mon-app.yaml  # Supprime et recrée
```

## Raccourcis et Alias Utiles

### 17. Alias Recommandés

```bash
# === Ajouter à ~/.bashrc ou ~/.zshrc ===

# Alias de base
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias kdes='kubectl describe'
alias klog='kubectl logs'
alias kex='kubectl exec -it'

# Alias avec namespace
alias kgpa='kubectl get pods --all-namespaces'
alias kgsa='kubectl get services --all-namespaces'

# Alias pour namespace système
alias ksys='kubectl -n kube-system'
alias kgsys='kubectl get -n kube-system'

# Changement rapide de namespace
alias kns='kubectl config set-context --current --namespace'

# Watch
alias kwatch='kubectl get pods -w'

# Suppression rapide
alias kdelp='kubectl delete pod'
alias kdeld='kubectl delete deployment'
alias kdels='kubectl delete service'

# Port-forward rapide
alias kpf='kubectl port-forward'

# Logs avec follow
alias klf='kubectl logs -f'

# Top
alias ktop='kubectl top'
alias ktopp='kubectl top pods'
alias ktopn='kubectl top nodes'
```

### 18. Autocomplétion

```bash
# === Configuration de l'autocomplétion ===

# Bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Avec alias
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc

# Zsh
source <(kubectl completion zsh)
echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)" >> ~/.zshrc

# Fish
kubectl completion fish | source
kubectl completion fish > ~/.config/fish/completions/kubectl.fish
```

## Commandes de Dépannage Rapide

### 19. Diagnostic Rapide

```bash
# === Check-list de dépannage ===

# 1. État général du cluster
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running

# 2. Événements récents
kubectl get events --sort-by='.lastTimestamp' | tail -20

# 3. Pods en erreur
kubectl get pods --all-namespaces | grep -E 'Error|CrashLoop|Pending'

# 4. Ressources d'un pod problématique
kubectl describe pod POD_NAME
kubectl logs POD_NAME --previous
kubectl get pod POD_NAME -o yaml | grep -A5 "status:"

# 5. Vérifier les services
kubectl get endpoints
kubectl get svc -o wide

# 6. Vérifier les ressources disponibles
kubectl top nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# 7. DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

### 20. Scripts Utiles

```bash
# === Script : Nettoyer les pods terminés ===
#!/bin/bash
kubectl delete pods --field-selector=status.phase=Succeeded --all-namespaces
kubectl delete pods --field-selector=status.phase=Failed --all-namespaces

# === Script : Redémarrer tous les deployments d'un namespace ===
#!/bin/bash
NAMESPACE=${1:-default}
kubectl get deployments -n $NAMESPACE -o name | xargs -I {} kubectl rollout restart {} -n $NAMESPACE

# === Script : Backup de toutes les ressources ===
#!/bin/bash
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get all -n $ns -o yaml > backup-$ns.yaml
done

# === Script : Watch pods avec couleurs ===
#!/bin/bash
watch -c "kubectl get pods --all-namespaces | grep -E --color=always 'Running|Error|CrashLoop|Pending|NAME'"

# === Script : Trouver les pods gourmands ===
#!/bin/bash
echo "Top 10 pods par CPU:"
kubectl top pods --all-namespaces | sort -k3 -rn | head -10
echo ""
echo "Top 10 pods par Mémoire:"
kubectl top pods --all-namespaces | sort -k4 -rn | head -10
```

## Aide-Mémoire par Cas d'Usage

### Déployer une Application

```bash
# 1. Créer le deployment
kubectl create deployment mon-app --image=mon-image:1.0

# 2. Exposer avec un service
kubectl expose deployment mon-app --port=80 --type=ClusterIP

# 3. Créer un ingress (depuis fichier)
kubectl apply -f ingress.yaml

# 4. Vérifier le déploiement
kubectl rollout status deployment/mon-app
kubectl get pods -l app=mon-app
```

### Déboguer une Application

```bash
# 1. Vérifier l'état
kubectl get pod mon-pod
kubectl describe pod mon-pod

# 2. Voir les logs
kubectl logs mon-pod -f

# 3. Exécuter des commandes
kubectl exec -it mon-pod -- /bin/bash

# 4. Port-forward pour test local
kubectl port-forward mon-pod 8080:80

# 5. Vérifier les événements
kubectl get events --field-selector involvedObject.name=mon-pod
```

### Mettre à Jour une Application

```bash
# 1. Mettre à jour l'image
kubectl set image deployment/mon-app mon-app=mon-image:2.0

# 2. Suivre le rollout
kubectl rollout status deployment/mon-app

# 3. Vérifier l'historique
kubectl rollout history deployment/mon-app

# 4. Si problème, rollback
kubectl rollout undo deployment/mon-app
```

### Scaler une Application

```bash
# 1. Scale manuel
kubectl scale deployment mon-app --replicas=5

# 2. Autoscale
kubectl autoscale deployment mon-app --min=2 --max=10 --cpu-percent=80

# 3. Vérifier
kubectl get hpa
kubectl get deployment mon-app
```

## Résolution de Problèmes Courants

### Pods en CrashLoopBackOff

```bash
# Diagnostic
kubectl describe pod POD_NAME          # Voir les événements
kubectl logs POD_NAME --previous       # Logs avant crash
kubectl get pod POD_NAME -o yaml | grep -A10 "lastState:"

# Solutions communes
kubectl delete pod POD_NAME            # Recréer le pod
kubectl edit deployment DEPLOY_NAME    # Modifier config
kubectl set env deployment/DEPLOY_NAME VAR=value  # Variables d'env
```

### Pods en Pending

```bash
# Diagnostic
kubectl describe pod POD_NAME          # Voir pourquoi pending
kubectl get nodes                      # Vérifier les nodes
kubectl top nodes                      # Ressources disponibles
kubectl get pvc                        # Volumes disponibles

# Solutions
kubectl describe nodes | grep -A5 "Allocated"  # Vérifier ressources
kubectl edit deployment DEPLOY_NAME    # Réduire requests/limits
```

### ImagePullBackOff

```bash
# Diagnostic
kubectl describe pod POD_NAME | grep -A5 "Events:"
kubectl get pod POD_NAME -o yaml | grep "image:"

# Solutions
# Vérifier le nom de l'image
kubectl edit deployment DEPLOY_NAME

# Pour registry privé
kubectl create secret docker-registry regcred \
  --docker-server=REGISTRY \
  --docker-username=USER \
  --docker-password=PASS

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

## Tips et Astuces

### Productivité

```bash
# Utiliser watch pour monitoring temps réel
watch kubectl get pods

# Utiliser grep pour filtrer
kubectl get pods --all-namespaces | grep Error

# Exporter en YAML pour backup
kubectl get deployment mon-app -o yaml > backup.yaml

# Appliquer récursivement
kubectl apply -R -f ./k8s/

# Attendre qu'un pod soit prêt
kubectl wait --for=condition=ready pod -l app=mon-app

# Utiliser xargs pour opérations en masse
kubectl get pods -o name | xargs -I {} kubectl delete {}
```

### Sécurité

```bash
# Voir qui peut faire quoi
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i --list

# Vérifier comme autre utilisateur
kubectl auth can-i create pods --as=john

# Voir les secrets (sans les valeurs)
kubectl get secrets
kubectl describe secret mon-secret  # Ne montre pas les valeurs
```

### Performance

```bash
# Limiter la sortie
kubectl get pods --no-headers          # Sans en-têtes
kubectl get pods -o name               # Juste les noms

# Cache local pour rapidité
kubectl proxy &                        # Démarrer proxy
curl localhost:8001/api/v1/namespaces/default/pods

# Batch operations
kubectl delete pods -l app=test --wait=false  # Pas attendre
```

## Conclusion

Cette collection de commandes kubectl constitue votre boîte à outils essentielle pour gérer efficacement votre cluster MicroK8s. Points clés à retenir :

1. **Commencez par les bases** : get, describe, logs, exec
2. **Utilisez les alias** : Gagnez du temps avec k, kgp, kgs
3. **Maîtrisez les options** : -o wide, -f, --all-namespaces
4. **Automatisez** : Scripts pour tâches répétitives
5. **Pratiquez le dépannage** : describe et logs sont vos amis

La maîtrise de kubectl vient avec la pratique. N'hésitez pas à expérimenter dans votre environnement de lab, utiliser `--dry-run` pour tester, et consulter `kubectl --help` pour découvrir de nouvelles options.

---

*Note : Cette référence est optimisée pour MicroK8s. Certaines commandes peuvent nécessiter des ajustements pour d'autres distributions Kubernetes. Utilisez toujours `microk8s kubectl` ou configurez l'alias approprié.*

⏭️
