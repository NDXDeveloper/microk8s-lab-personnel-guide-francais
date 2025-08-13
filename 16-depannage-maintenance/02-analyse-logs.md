🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.2 Analyse des logs

## Introduction à l'importance des logs

Les logs sont la boîte noire de votre cluster Kubernetes. Ils enregistrent tout ce qui se passe : démarrages d'applications, erreurs, requêtes traitées, problèmes de connexion, et bien plus encore. Savoir où trouver les logs, comment les lire et les interpréter est une compétence fondamentale pour diagnostiquer efficacement les problèmes dans MicroK8s. Cette section vous guidera à travers les différents types de logs, leurs emplacements, et les techniques pour les analyser efficacement.

## Architecture des logs dans Kubernetes

### Comprendre les niveaux de logs

Dans un cluster MicroK8s, les logs existent à plusieurs niveaux, chacun fournissant des informations différentes et complémentaires :

**Logs d'application** : Ce sont les logs générés par vos applications déployées dans les pods. Ils contiennent les messages que votre application écrit sur stdout (sortie standard) et stderr (erreur standard). Ces logs sont généralement les premiers à consulter quand une application ne fonctionne pas correctement.

**Logs de conteneur** : Le runtime des conteneurs (containerd dans MicroK8s) génère ses propres logs concernant le cycle de vie des conteneurs : création, démarrage, arrêt, erreurs de montage de volumes, etc.

**Logs Kubernetes** : Les composants de Kubernetes eux-mêmes (API server, scheduler, controller manager, kubelet) produisent des logs détaillant leurs opérations. Dans MicroK8s, ces composants sont intégrés dans le processus kubelite.

**Logs système** : Au niveau du système d'exploitation, systemd (ou autre système d'init) capture les logs des services, incluant MicroK8s lui-même.

### Cycle de vie et rétention des logs

Il est important de comprendre que les logs dans Kubernetes ne sont pas éternels. Par défaut :

- Les logs des pods sont conservés tant que le pod existe
- Quand un conteneur redémarre, les logs précédents sont accessibles avec l'option `--previous`
- Quand un pod est supprimé, ses logs disparaissent également
- Les logs système sont soumis à la rotation configurée par votre système (généralement via logrotate ou journald)

Cette nature éphémère des logs souligne l'importance d'une stratégie de centralisation des logs pour les environnements de production, même si pour un lab personnel, les logs locaux sont souvent suffisants.

## Accès aux logs des applications

### Logs basiques des pods

La commande la plus fondamentale pour accéder aux logs d'une application est :

```bash
microk8s kubectl logs <nom-du-pod> -n <namespace>
```

Cette commande simple cache plusieurs subtilités importantes. Par défaut, elle affiche les logs du conteneur principal du pod. Si votre pod contient plusieurs conteneurs (par exemple, un conteneur d'application et un sidecar pour la collecte de métriques), vous devez spécifier le conteneur :

```bash
microk8s kubectl logs <nom-du-pod> -c <nom-du-conteneur> -n <namespace>
```

### Suivre les logs en temps réel

Pour surveiller une application en cours d'exécution, l'option `-f` (follow) est essentielle :

```bash
microk8s kubectl logs -f <nom-du-pod> -n <namespace>
```

Cette commande fonctionne comme `tail -f` sous Linux : elle affiche les nouvelles lignes de log au fur et à mesure qu'elles sont générées. C'est particulièrement utile pour :
- Surveiller le démarrage d'une application
- Observer le comportement lors de tests
- Diagnostiquer des problèmes intermittents

Pour arrêter le suivi, utilisez Ctrl+C.

### Gestion du volume de logs

Les applications peuvent générer énormément de logs. Pour éviter d'être submergé, plusieurs options sont disponibles :

```bash
# Afficher seulement les 100 dernières lignes
microk8s kubectl logs <nom-du-pod> -n <namespace> --tail=100

# Afficher les logs depuis les 5 dernières minutes
microk8s kubectl logs <nom-du-pod> -n <namespace> --since=5m

# Afficher les logs depuis une heure spécifique
microk8s kubectl logs <nom-du-pod> -n <namespace> --since-time=2024-01-15T10:00:00Z

# Combiner les options pour plus de précision
microk8s kubectl logs <nom-du-pod> -n <namespace> --tail=50 --since=10m
```

### Logs de pods avec plusieurs réplicas

Pour les déploiements avec plusieurs réplicas, récupérer les logs de tous les pods peut être fastidieux. Utilisez les labels pour simplifier :

```bash
# Logs de tous les pods avec un label spécifique
microk8s kubectl logs -l app=nginx -n <namespace>

# Suivre les logs de tous les pods d'un déploiement
microk8s kubectl logs -f -l app=nginx -n <namespace> --prefix=true
```

L'option `--prefix=true` ajoute le nom du pod avant chaque ligne de log, facilitant l'identification de la source quand vous surveillez plusieurs pods simultanément.

### Logs des conteneurs redémarrés

Quand un conteneur crash et redémarre, les logs du crash sont cruciaux pour comprendre le problème :

```bash
# Voir les logs du conteneur précédent (avant le dernier redémarrage)
microk8s kubectl logs <nom-du-pod> -n <namespace> --previous

# Si plusieurs conteneurs, spécifier lequel
microk8s kubectl logs <nom-du-pod> -c <conteneur> -n <namespace> --previous
```

Cette commande est vitale quand vous voyez un pod avec un statut "CrashLoopBackOff" - elle vous permet de voir pourquoi le conteneur crash avant qu'il ne redémarre.

## Logs des composants Kubernetes

### Logs du système MicroK8s

MicroK8s utilise systemd pour gérer ses services. Pour accéder aux logs système de MicroK8s :

```bash
# Logs du daemon principal MicroK8s
journalctl -u snap.microk8s.daemon-kubelite -f

# Voir les logs des dernières 2 heures
journalctl -u snap.microk8s.daemon-kubelite --since "2 hours ago"

# Logs avec plus de détails (verbosité augmentée)
journalctl -u snap.microk8s.daemon-kubelite -xe

# Filtrer par priorité (erreurs seulement)
journalctl -u snap.microk8s.daemon-kubelite -p err
```

### Logs des composants du control plane

Dans MicroK8s, les composants du control plane sont intégrés, mais vous pouvez toujours accéder à leurs logs spécifiques :

```bash
# Logs de l'API server (intégré dans kubelite)
journalctl -u snap.microk8s.daemon-kubelite | grep apiserver

# Logs du scheduler
journalctl -u snap.microk8s.daemon-kubelite | grep scheduler

# Logs du controller manager
journalctl -u snap.microk8s.daemon-kubelite | grep controller
```

### Logs des addons système

Les addons comme CoreDNS, ingress controller, ou dashboard s'exécutent comme des pods dans le namespace `kube-system` :

```bash
# Logs de CoreDNS
microk8s kubectl logs -n kube-system -l k8s-app=kube-dns

# Logs de l'ingress controller
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s

# Logs du dashboard
microk8s kubectl logs -n kube-system -l k8s-app=kubernetes-dashboard
```

## Interprétation des logs

### Structure typique des logs

Les logs Kubernetes suivent généralement un format structuré. Voici les éléments communs à rechercher :

**Timestamp** : La plupart des logs commencent par un horodatage. Pour les afficher même si l'application ne les inclut pas :

```bash
microk8s kubectl logs <pod> --timestamps
```

**Niveau de log** : Les niveaux standards sont :
- `DEBUG` : Informations détaillées pour le débogage
- `INFO` : Informations générales sur le fonctionnement normal
- `WARNING`/`WARN` : Situations inhabituelles mais non critiques
- `ERROR` : Erreurs qui nécessitent attention
- `FATAL`/`CRITICAL` : Erreurs graves causant l'arrêt de l'application

**Message** : Le contenu principal décrivant l'événement

**Contexte** : Informations additionnelles comme l'ID de requête, l'utilisateur, etc.

### Patterns d'erreurs courants

Reconnaître les patterns d'erreurs courants accélère considérablement le diagnostic. Voici les plus fréquents :

**Erreurs de connexion réseau** :
```
dial tcp: lookup <service>: no such host
connection refused
timeout waiting for connection
network is unreachable
```
Ces erreurs indiquent généralement des problèmes DNS, des services non démarrés, ou des problèmes de configuration réseau.

**Erreurs de permissions** :
```
permission denied
forbidden: User cannot
cannot create resource
operation not permitted
```
Ces messages suggèrent des problèmes RBAC ou des permissions système insuffisantes.

**Erreurs de ressources** :
```
OOMKilled (Out Of Memory)
insufficient cpu
disk pressure
cannot allocate memory
```
Ces erreurs indiquent que les limites de ressources sont atteintes ou mal configurées.

**Erreurs de configuration** :
```
invalid configuration
missing required field
unknown field
validation error
```
Ces messages pointent vers des erreurs dans les manifestes YAML ou les ConfigMaps.

### Techniques de filtrage et recherche

Pour analyser efficacement de gros volumes de logs, maîtrisez les outils de filtrage :

```bash
# Rechercher un terme spécifique
microk8s kubectl logs <pod> | grep ERROR

# Recherche insensible à la casse
microk8s kubectl logs <pod> | grep -i error

# Exclure certaines lignes
microk8s kubectl logs <pod> | grep -v DEBUG

# Recherche avec contexte (3 lignes avant et après)
microk8s kubectl logs <pod> | grep -C 3 "connection refused"

# Compter les occurrences
microk8s kubectl logs <pod> | grep -c ERROR

# Utiliser awk pour des analyses plus complexes
microk8s kubectl logs <pod> | awk '/ERROR/ {print $1, $2, $NF}'
```

## Logs avancés et debugging

### Augmenter la verbosité

Pour obtenir plus d'informations lors du debugging, vous pouvez augmenter la verbosité des commandes kubectl :

```bash
# Niveau de verbosité de 0 à 9
microk8s kubectl get pods -v=6

# Très verbeux (niveau 9) - affiche les requêtes HTTP
microk8s kubectl get pods -v=9
```

Cette technique est utile pour diagnostiquer les problèmes de communication avec l'API server.

### Logs d'événements Kubernetes

Les événements Kubernetes complètent les logs en fournissant une vue de haut niveau des actions dans le cluster :

```bash
# Événements récents triés par timestamp
microk8s kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Événements pour un pod spécifique
microk8s kubectl get events --field-selector involvedObject.name=<pod-name>

# Surveiller les événements en temps réel
microk8s kubectl get events -w --all-namespaces
```

Les événements sont particulièrement utiles pour comprendre pourquoi un pod ne démarre pas ou pourquoi un volume ne se monte pas.

### Debugging avec ephemeral containers

Pour les cas complexes, vous pouvez attacher un conteneur de debug à un pod en cours d'exécution :

```bash
# Attacher un conteneur de debug
microk8s kubectl debug <pod-name> -it --image=busybox

# Avec des outils réseau
microk8s kubectl debug <pod-name> -it --image=nicolaka/netshoot
```

Cette technique permet d'inspecter l'environnement du pod sans modifier l'application.

## Organisation et gestion des logs

### Sauvegarde des logs pour analyse

Pour une analyse approfondie ou pour partager avec d'autres, sauvegardez les logs :

```bash
# Sauvegarder dans un fichier
microk8s kubectl logs <pod> > pod_logs_$(date +%Y%m%d_%H%M%S).txt

# Sauvegarder avec métadonnées
echo "=== Pod Description ===" > diagnostic.txt
microk8s kubectl describe pod <pod> >> diagnostic.txt
echo -e "\n=== Pod Logs ===" >> diagnostic.txt
microk8s kubectl logs <pod> >> diagnostic.txt
```

### Rotation et nettoyage des logs

Pour éviter que les logs système ne remplissent votre disque :

```bash
# Vérifier l'espace utilisé par journald
journalctl --disk-usage

# Nettoyer les logs de plus de 7 jours
sudo journalctl --vacuum-time=7d

# Limiter la taille totale des logs
sudo journalctl --vacuum-size=1G

# Configuration permanente dans /etc/systemd/journald.conf
# SystemMaxUse=1G
# MaxRetentionSec=1week
```

### Scripts d'analyse automatisée

Créez des scripts pour automatiser l'analyse des patterns courants :

```bash
#!/bin/bash
# Script d'analyse des erreurs

NAMESPACE=${1:-default}
SINCE=${2:-1h}

echo "Analyse des erreurs dans le namespace $NAMESPACE (dernières $SINCE)"
echo "================================================"

echo -e "\n### Pods en erreur ###"
microk8s kubectl get pods -n $NAMESPACE | grep -v Running | grep -v Completed

echo -e "\n### Résumé des erreurs par pod ###"
for pod in $(microk8s kubectl get pods -n $NAMESPACE -o name); do
    pod_name=$(echo $pod | cut -d'/' -f2)
    error_count=$(microk8s kubectl logs $pod_name -n $NAMESPACE --since=$SINCE 2>/dev/null | grep -c ERROR)
    if [ $error_count -gt 0 ]; then
        echo "$pod_name: $error_count erreurs"
    fi
done

echo -e "\n### Top 10 des messages d'erreur ###"
microk8s kubectl logs --all-containers=true -n $NAMESPACE --since=$SINCE 2>/dev/null | \
    grep ERROR | \
    sed 's/^[^ ]* [^ ]* //' | \
    sort | uniq -c | sort -rn | head -10
```

## Corrélation des logs

### Chronologie des événements

Pour comprendre une séquence d'événements, synchronisez les timestamps :

```bash
# Combiner logs et événements avec timestamps
(
    echo "=== EVENTS ==="
    microk8s kubectl get events --sort-by='.lastTimestamp' | tail -20
    echo -e "\n=== POD LOGS ==="
    microk8s kubectl logs <pod> --timestamps --tail=50
) | sort -k1,2
```

### Traçage des requêtes

Pour suivre une requête à travers plusieurs services, recherchez des identifiants communs :

```bash
# Si vos apps utilisent des request IDs
REQUEST_ID="abc-123-def"
for pod in $(microk8s kubectl get pods -o name); do
    echo "Checking $pod..."
    microk8s kubectl logs $pod | grep $REQUEST_ID
done
```

### Logs multi-composants

Pour des problèmes complexes impliquant plusieurs composants :

```bash
# Créer une vue consolidée
{
    echo "=== Ingress Controller ==="
    microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s --tail=20
    echo -e "\n=== Application ==="
    microk8s kubectl logs -l app=myapp --tail=20
    echo -e "\n=== Database ==="
    microk8s kubectl logs -l app=postgres --tail=20
} > consolidated_logs.txt
```

## Outils et techniques complémentaires

### Utilisation de stern pour les logs multi-pods

Stern est un outil tiers très utile pour suivre les logs de plusieurs pods :

```bash
# Installation
sudo snap install stern

# Suivre tous les pods d'un namespace
stern . -n <namespace>

# Suivre avec un pattern
stern "nginx-*" --since 15m

# Avec coloration par pod
stern . -n <namespace> --color always
```

### Analyse avec jq pour les logs JSON

Si vos applications produisent des logs JSON :

```bash
# Parser et formater les logs JSON
microk8s kubectl logs <pod> | jq '.'

# Filtrer par niveau
microk8s kubectl logs <pod> | jq 'select(.level == "error")'

# Extraire des champs spécifiques
microk8s kubectl logs <pod> | jq '{time: .timestamp, msg: .message, level: .level}'

# Compter par type d'erreur
microk8s kubectl logs <pod> | jq -r '.error_type' | sort | uniq -c
```

## Bonnes pratiques pour les logs

### Configuration des applications

Pour faciliter l'analyse des logs, configurez vos applications pour :

1. **Utiliser des niveaux de log cohérents** : Adoptez une convention (DEBUG, INFO, WARN, ERROR) et respectez-la
2. **Inclure des timestamps** : Même si Kubernetes peut les ajouter, c'est mieux dans l'application
3. **Structurer les logs** : Préférez JSON ou un format structuré au texte libre
4. **Inclure le contexte** : User ID, request ID, transaction ID facilitent le traçage
5. **Éviter les logs sensibles** : Pas de mots de passe, tokens, ou données personnelles

### Stratégie de rétention

Pour un lab personnel, une stratégie simple mais efficace :

1. **Logs système** : 7-30 jours selon l'espace disponible
2. **Logs d'application** : Exporter les logs importants avant suppression des pods
3. **Logs de debug** : Activer temporairement, désactiver après résolution
4. **Archives** : Compresser et archiver les logs d'incidents pour référence future

## Résolution de problèmes courants via les logs

### Pod en CrashLoopBackOff

Quand un pod redémarre continuellement :

```bash
# 1. Vérifier le statut détaillé
microk8s kubectl describe pod <pod-name>

# 2. Voir les logs du dernier crash
microk8s kubectl logs <pod-name> --previous

# 3. Chercher les patterns d'erreur
microk8s kubectl logs <pod-name> --previous | grep -E "ERROR|FATAL|Exception|Error:"

# 4. Vérifier les événements
microk8s kubectl get events --field-selector involvedObject.name=<pod-name>
```

### Service inaccessible

Pour diagnostiquer pourquoi un service ne répond pas :

```bash
# 1. Vérifier que les pods backend sont running
microk8s kubectl get endpoints <service-name>

# 2. Logs de l'ingress controller
microk8s kubectl logs -n ingress -l name=nginx-ingress-microk8s | grep <service-name>

# 3. Logs des pods du service
microk8s kubectl logs -l app=<app-label>

# 4. Tester depuis un pod de debug
microk8s kubectl run debug --image=nicolaka/netshoot --rm -it -- /bin/bash
# Dans le pod: curl http://<service-name>.<namespace>.svc.cluster.local
```

### Problèmes de performance

Pour identifier les goulots d'étranglement :

```bash
# 1. Analyser les temps de réponse dans les logs
microk8s kubectl logs <pod> | grep "response_time" | awk '{print $NF}' | sort -n | tail -20

# 2. Chercher les timeouts
microk8s kubectl logs <pod> | grep -i timeout

# 3. Vérifier les erreurs de ressources
microk8s kubectl describe pod <pod> | grep -A 5 "Events:"
```

## Conclusion

L'analyse des logs est un art qui s'affine avec la pratique. Les logs sont votre fenêtre sur le comportement interne de votre cluster et de vos applications. En maîtrisant les techniques présentées dans cette section, vous serez capable de diagnostiquer la grande majorité des problèmes que vous rencontrerez. Rappelez-vous que les logs racontent une histoire - votre rôle est d'apprendre à la lire efficacement. La section suivante sur les problèmes réseau courants s'appuiera sur ces compétences d'analyse des logs pour résoudre une catégorie spécifique mais fréquente de problèmes.

⏭️
