🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 Pod Security Standards

## Introduction aux Pod Security Standards

Imaginez que vous gérez un immeuble d'appartements (votre cluster Kubernetes). Les Network Policies contrôlent qui peut entrer dans l'immeuble et circuler entre les appartements. Les Pod Security Standards (PSS), eux, définissent ce que les locataires peuvent faire à l'intérieur de leur appartement : peuvent-ils percer les murs ? Installer un coffre-fort ? Accéder aux conduits de ventilation ?

Les Pod Security Standards établissent des règles sur ce qu'un pod peut ou ne peut pas faire une fois qu'il s'exécute dans votre cluster, limitant ses capacités pour éviter qu'un pod compromis ne puisse escalader ses privilèges ou affecter le système hôte.

### Pourquoi les Pod Security Standards sont essentiels

Sans restrictions, un conteneur peut :
- **Monter le système de fichiers de l'hôte** et lire tous les secrets
- **S'exécuter en tant que root** avec tous les privilèges
- **Modifier les paramètres kernel** du nœud
- **Accéder aux périphériques** physiques
- **Échapper au conteneur** et compromettre l'hôte

Les PSS empêchent ces comportements dangereux en appliquant des politiques de sécurité au moment de la création des pods.

## Évolution : De PSP à PSS

### L'histoire des Pod Security Policies (PSP)

Kubernetes a d'abord introduit les Pod Security Policies (PSP), mais elles étaient :
- Complexes à configurer correctement
- Difficiles à déboguer
- Source de nombreuses erreurs de configuration

Les PSP ont été dépréciées dans Kubernetes 1.21 et supprimées dans la version 1.25.

### Les nouveaux Pod Security Standards (PSS)

Les PSS sont la nouvelle approche, plus simple et plus robuste :
- **Trois niveaux prédéfinis** au lieu de politiques personnalisées complexes
- **Application au niveau du namespace** pour plus de clarté
- **Modes d'application flexibles** : warn, audit, ou enforce
- **Migration progressive** possible depuis les anciennes configurations

## Les trois niveaux de sécurité

### 1. Privileged (Privilégié) - Aucune restriction

Le niveau **Privileged** n'applique aucune restriction. C'est l'équivalent de donner les clés de tout l'immeuble à chaque locataire.

**Utilisation appropriée :**
- Composants système critiques (CNI, CSI, monitoring)
- Environnements de développement isolés
- Outils d'administration du cluster

**Risques :**
- Accès complet au système hôte
- Possibilité d'escalade de privilèges
- Aucune isolation de sécurité

### 2. Baseline (Base) - Restrictions minimales

Le niveau **Baseline** empêche les escalades de privilèges connues tout en restant permissif pour la plupart des applications.

**Ce qui est bloqué :**
- Privilèges élevés (`privileged: true`)
- Montage de volumes hostPath sensibles
- Capacités Linux dangereuses
- Profils AppArmor ou SELinux personnalisés

**Ce qui est autorisé :**
- Exécution en tant que root (mais sans privilèges)
- La plupart des types de volumes
- Ports privilégiés (< 1024)

### 3. Restricted (Restreint) - Sécurité maximale

Le niveau **Restricted** applique les meilleures pratiques de sécurité actuelles. C'est le niveau recommandé pour les applications en production.

**Restrictions supplémentaires :**
- Interdiction de s'exécuter en tant que root
- Obligation d'avoir un système de fichiers root en lecture seule
- Interdiction d'escalade de privilèges
- Limitation stricte des capacités Linux
- Restrictions sur les types de volumes

## Configuration dans MicroK8s

### Activation des Pod Security Standards

Dans MicroK8s, les PSS peuvent être configurés au niveau du namespace :

```bash
# Vérifier la version de Kubernetes (doit être >= 1.23)
microk8s kubectl version

# Les PSS sont activés par défaut dans les versions récentes
# Vérifier les labels sur un namespace
microk8s kubectl describe namespace default | grep "pod-security"
```

### Application des standards à un namespace

Les PSS utilisent des labels de namespace pour définir les politiques :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce le niveau restricted
    pod-security.kubernetes.io/enforce: restricted
    # Version du standard à utiliser
    pod-security.kubernetes.io/enforce-version: latest

    # Warn pour le niveau restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest

    # Audit pour le niveau restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Les trois modes d'application

#### Mode Enforce (Blocage)
```yaml
pod-security.kubernetes.io/enforce: restricted
```
- **Bloque** la création de pods non conformes
- Protection immédiate
- Peut casser des déploiements existants

#### Mode Warn (Avertissement)
```yaml
pod-security.kubernetes.io/warn: restricted
```
- **Avertit** lors de la création de pods non conformes
- Le pod est quand même créé
- Utile pour identifier les problèmes

#### Mode Audit (Audit)
```yaml
pod-security.kubernetes.io/audit: restricted
```
- **Enregistre** les violations dans les logs d'audit
- Aucun impact visible pour l'utilisateur
- Parfait pour analyser l'impact avant migration

## Exemples pratiques par niveau

### Exemple Privileged : Composant système

```yaml
# Namespace pour composants système
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
# Pod privilégié (accepté dans ce namespace)
apiVersion: v1
kind: Pod
metadata:
  name: system-debugger
  namespace: kube-system
spec:
  containers:
  - name: debugger
    image: nicolaka/netshoot
    securityContext:
      privileged: true  # Autorisé en mode privileged
      runAsUser: 0      # Root autorisé
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /        # Montage du système hôte autorisé
```

### Exemple Baseline : Application standard

```yaml
# Namespace avec niveau baseline
apiVersion: v1
kind: Namespace
metadata:
  name: applications
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
---
# Pod conforme au niveau baseline
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: applications
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    securityContext:
      runAsUser: 0           # Root autorisé en baseline
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE   # Pour bind sur port 80
```

### Exemple Restricted : Application sécurisée

```yaml
# Namespace avec niveau restricted
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
# Pod conforme au niveau restricted
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure-apps
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080    # Port non privilégié
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

## SecurityContext : Configuration de sécurité des pods

### Niveau Pod vs Niveau Container

Le SecurityContext peut être défini à deux niveaux :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  # SecurityContext au niveau du Pod (s'applique à tous les conteneurs)
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]

  containers:
  - name: container1
    image: busybox
    # SecurityContext au niveau du conteneur (override le pod)
    securityContext:
      runAsUser: 2000  # Override le 1000 du pod
      capabilities:
        add: ["NET_ADMIN"]
```

### Options principales du SecurityContext

#### Gestion des utilisateurs et groupes

```yaml
securityContext:
  runAsUser: 1000        # UID de l'utilisateur
  runAsGroup: 3000       # GID principal
  runAsNonRoot: true     # Interdire root (UID 0)
  fsGroup: 2000          # GID pour les volumes
  supplementalGroups:    # Groupes supplémentaires
  - 4000
  - 5000
```

#### Système de fichiers

```yaml
securityContext:
  readOnlyRootFilesystem: true  # Root filesystem en lecture seule
  fsGroupChangePolicy: "OnRootMismatch"  # Quand changer les permissions
```

#### Privilèges et capacités

```yaml
securityContext:
  privileged: false              # Mode privilégié (dangereux)
  allowPrivilegeEscalation: false # Empêcher l'escalade
  capabilities:
    drop:
    - ALL                        # Supprimer toutes les capacités
    add:
    - NET_BIND_SERVICE          # Ajouter seulement celles nécessaires
```

#### Profils de sécurité

```yaml
securityContext:
  seLinuxOptions:
    user: "system_u"
    role: "system_r"
    type: "svirt_lxc_net_t"
    level: "s0:c123,c456"
  seccompProfile:
    type: RuntimeDefault         # ou Localhost, Unconfined
  appArmorProfile:
    type: RuntimeDefault
```

## Capacités Linux expliquées

Les capacités Linux permettent de donner des privilèges spécifiques sans donner root complet :

### Capacités courantes et leurs usages

```yaml
capabilities:
  drop:
  - ALL  # Toujours commencer par tout supprimer
  add:
  # Capacités réseau
  - NET_BIND_SERVICE    # Bind sur ports < 1024
  - NET_ADMIN          # Configuration réseau
  - NET_RAW            # Sockets raw (ping)

  # Capacités système
  - SYS_TIME           # Modifier l'horloge système
  - SYS_PTRACE         # Débugger d'autres processus
  - SYS_ADMIN          # Opérations d'administration (DANGEREUX)

  # Capacités fichiers
  - CHOWN              # Changer les propriétaires de fichiers
  - DAC_OVERRIDE       # Ignorer les permissions de fichiers
  - FOWNER             # Opérations sur les fichiers
  - SETUID             # Changer l'UID
  - SETGID             # Changer le GID

  # Autres
  - KILL               # Envoyer des signaux à n'importe quel processus
  - AUDIT_WRITE        # Écrire dans les logs d'audit
```

### Exemple : Application web avec capacités minimales

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Si besoin du port 80
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-cache
          mountPath: /var/cache/nginx
        - name: var-run
          mountPath: /var/run
      volumes:
      - name: tmp
        emptyDir: {}
      - name: var-cache
        emptyDir: {}
      - name: var-run
        emptyDir: {}
```

## Migration vers Pod Security Standards

### Étape 1 : Audit de l'existant

```bash
# Créer un namespace de test avec audit
cat <<EOF | microk8s kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: migration-test
  labels:
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Déployer vos applications dans ce namespace
microk8s kubectl apply -f your-app.yaml -n migration-test

# Vérifier les warnings
microk8s kubectl get events -n migration-test | grep Warning
```

### Étape 2 : Identifier les violations

```bash
# Script pour tester la conformité d'un pod
cat <<'EOF' > check-pod-security.sh
#!/bin/bash
NAMESPACE=$1
POD=$2

echo "Checking pod $POD in namespace $NAMESPACE"

# Récupérer la config du pod
microk8s kubectl get pod $POD -n $NAMESPACE -o yaml > /tmp/pod.yaml

# Vérifications pour Restricted
echo "=== Restricted Level Checks ==="

# Check runAsNonRoot
if grep -q "runAsUser: 0" /tmp/pod.yaml; then
  echo "❌ Running as root (runAsUser: 0)"
else
  echo "✅ Not running as root"
fi

# Check privileged
if grep -q "privileged: true" /tmp/pod.yaml; then
  echo "❌ Privileged mode enabled"
else
  echo "✅ Privileged mode disabled"
fi

# Check allowPrivilegeEscalation
if grep -q "allowPrivilegeEscalation: true" /tmp/pod.yaml; then
  echo "❌ Privilege escalation allowed"
else
  echo "✅ Privilege escalation disabled"
fi

# Check capabilities
if grep -A5 "capabilities:" /tmp/pod.yaml | grep -q "add:"; then
  echo "⚠️  Additional capabilities detected:"
  grep -A5 "capabilities:" /tmp/pod.yaml
else
  echo "✅ No additional capabilities"
fi

# Check volumes
if grep -q "hostPath:" /tmp/pod.yaml; then
  echo "❌ HostPath volumes detected"
else
  echo "✅ No hostPath volumes"
fi

rm /tmp/pod.yaml
EOF

chmod +x check-pod-security.sh
./check-pod-security.sh default my-pod
```

### Étape 3 : Corriger les violations

#### Violation : Running as root

**Problème :**
```yaml
# Non conforme
spec:
  containers:
  - name: app
    image: myapp:latest
    # Pas de securityContext ou runAsUser: 0
```

**Solution :**
```yaml
# Conforme
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
```

#### Violation : Système de fichiers writable

**Problème :**
```yaml
# Non conforme
spec:
  containers:
  - name: app
    image: myapp:latest
    # Système de fichiers root modifiable
```

**Solution :**
```yaml
# Conforme
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: app-data
      mountPath: /app/data
  volumes:
  - name: tmp
    emptyDir: {}
  - name: app-data
    emptyDir: {}
```

### Étape 4 : Application progressive

```yaml
# Phase 1 : Audit seulement
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/audit: restricted
---
# Phase 2 : Audit + Warning (après 1 semaine)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Phase 3 : Enforcement (après validation)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Patterns de configuration courants

### Pattern 1 : Application web stateless

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: webapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      volumes:
      - name: tmp
        emptyDir:
          sizeLimit: 1Gi
      - name: cache
        emptyDir:
          sizeLimit: 2Gi
```

### Pattern 2 : Base de données avec volumes persistants

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999  # User postgres
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: postgres
        image: postgres:14-alpine
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: tmp
          mountPath: /tmp
        - name: run
          mountPath: /var/run/postgresql
      volumes:
      - name: tmp
        emptyDir: {}
      - name: run
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### Pattern 3 : Job de maintenance

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
  namespace: maintenance
spec:
  template:
    spec:
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["/bin/sh"]
        args: ["-c", "backup-script.sh"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: backup-data
          mountPath: /backup
      volumes:
      - name: tmp
        emptyDir: {}
      - name: backup-data
        persistentVolumeClaim:
          claimName: backup-pvc
```

## Outils et commandes utiles

### Vérification des labels PSS

```bash
# Voir tous les namespaces avec leurs niveaux PSS
microk8s kubectl get namespaces -o json | jq -r '.items[] |
  {
    name: .metadata.name,
    enforce: .metadata.labels["pod-security.kubernetes.io/enforce"],
    warn: .metadata.labels["pod-security.kubernetes.io/warn"],
    audit: .metadata.labels["pod-security.kubernetes.io/audit"]
  }'

# Vérifier un namespace spécifique
microk8s kubectl get namespace production --show-labels
```

### Test de conformité

```bash
# Dry-run pour tester si un pod serait accepté
microk8s kubectl run test-pod --image=nginx --dry-run=server -o yaml

# Appliquer temporairement un niveau différent
microk8s kubectl label namespace test-ns \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite

# Voir les événements de violation
microk8s kubectl get events -A | grep -i "violates PodSecurity"
```

### Scripts d'aide

```bash
# Script pour appliquer PSS à tous les namespaces applicatifs
cat <<'EOF' > apply-pss-all.sh
#!/bin/bash
LEVEL=${1:-baseline}
MODE=${2:-warn}

for ns in $(microk8s kubectl get ns -o name | grep -v kube- | cut -d/ -f2); do
  echo "Applying $MODE=$LEVEL to namespace $ns"
  microk8s kubectl label namespace $ns \
    pod-security.kubernetes.io/$MODE=$LEVEL \
    --overwrite
done
EOF

chmod +x apply-pss-all.sh
./apply-pss-all.sh restricted audit
```

## Exceptions et cas spéciaux

### Exemption de namespaces système

Certains namespaces doivent rester en mode privileged :

```yaml
# Namespaces système typiques
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
apiVersion: v1
kind: Namespace
metadata:
  name: kube-public
  labels:
    pod-security.kubernetes.io/enforce: baseline
---
apiVersion: v1
kind: Namespace
metadata:
  name: kube-node-lease
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### Gestion des outils de monitoring

Les outils comme Prometheus nécessitent souvent des permissions spéciales :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    # Baseline car besoin d'accès aux métriques
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

### Applications legacy

Pour les applications qui ne peuvent pas être modifiées :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: legacy-apps
  labels:
    # Commencer permissif
    pod-security.kubernetes.io/enforce: privileged
    # Mais auditer pour restricted
    pod-security.kubernetes.io/audit: restricted
  annotations:
    migration-status: "pending"
    target-level: "restricted"
    review-date: "2024-06-01"
```

## Bonnes pratiques

### 1. Stratégie de namespace

```yaml
# Structure recommandée
production/
├── enforce: restricted
├── warn: restricted
└── audit: restricted

staging/
├── enforce: baseline
├── warn: restricted
└── audit: restricted

development/
├── enforce: baseline
├── warn: baseline
└── audit: restricted

system/
├── enforce: privileged
└── audit: baseline
```

### 2. Documentation des exceptions

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: special-app
  labels:
    pod-security.kubernetes.io/enforce: baseline
  annotations:
    security-exception: "true"
    exception-reason: "Requires NET_ADMIN for VPN functionality"
    exception-approved-by: "security-team"
    exception-review-date: "2024-12-31"
```

### 3. Templates de SecurityContext

```yaml
# Template pour applications web
webAppSecurityContext: &webAppSecurity
  runAsNonRoot: true
  runAsUser: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault

webAppContainerSecurity: &webAppContainerSecurity
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL

# Utilisation
spec:
  securityContext: *webAppSecurity
  containers:
  - name: app
    securityContext: *webAppContainerSecurity
```

### 4. Checklist de migration

- [ ] Inventaire des applications actuelles
- [ ] Classification par niveau de sécurité requis
- [ ] Test en mode audit sur environnement de test
- [ ] Correction des violations identifiées
- [ ] Test en mode warn sur staging
- [ ] Documentation des exceptions nécessaires
- [ ] Déploiement progressif en production
- [ ] Monitoring des violations post-déploiement

## Dépannage courant

### Problème : "Error creating pod: violates PodSecurity"

**Diagnostic :**
```bash
# Voir le niveau actuel du namespace
microk8s kubectl get ns <namespace> -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}'

# Voir les détails de l'erreur
microk8s kubectl describe pod <pod-name> -n <namespace>
```

**Solutions communes :**
1. Ajuster le SecurityContext du pod
2. Changer temporairement le niveau du namespace
3. Créer un namespace dédié avec niveau approprié

### Problème : Application ne démarre pas après sécurisation

**Causes fréquentes :**
- Besoin d'écrire dans des répertoires spécifiques
- Port privilégié nécessaire
- Dépendance sur l'utilisateur root

**Debug :**
```bash
# Lancer un shell de debug
microk8s kubectl run debug --rm -it --image=busybox \
  --overrides='{"spec":{"securityContext":{"runAsUser":1000}}}' \
  -- sh

# Vérifier les permissions
ls -la /
whoami
id
```

### Problème : Volumes non accessibles

**Solution avec fsGroup :**
```yaml
spec:
  securityContext:
    fsGroup: 2000  # Groupe qui possédera les volumes
    fsGroupChangePolicy: "OnRootMismatch"
```

## Monitoring et alerting

### Prometheus rules pour PSS

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-security-alerts
spec:
  groups:
  - name: pod-security
    rules:
    - alert: PodSecurityViolation
      expr: |
        rate(pod_security_evaluations_total{
          decision="deny"
        }[5m]) > 0
      annotations:
        summary: "Pod security violation in {{ $labels.namespace }}"
        description: "Pod creation denied due to PSS policy"

    - alert: PrivilegedPodCreated
      expr: |
        kube_pod_container_status_running{
          namespace!~"kube-.*"
        } * on(pod,namespace)
        kube_pod_container_info{
          container_security_context_privileged="true"
        } > 0
      annotations:
        summary: "Privileged pod running in {{ $labels.namespace }}"
```

## Résumé et points clés

Les Pod Security Standards sont votre dernière ligne de défense :

1. **Trois niveaux simples** : Privileged, Baseline, Restricted
2. **Application par namespace** : Plus clair que les anciennes PSP
3. **Migration progressive** : Audit → Warn → Enforce
4. **SecurityContext est votre ami** : Configurez-le correctement dès le début
5. **Principe du moindre privilège** : Commencez restrictif, relâchez si nécessaire
6. **Documentez les exceptions** : Toute déviation doit être justifiée
7. **Testez en amont** : Utilisez audit et warn avant enforce

Les PSS demandent un effort initial pour adapter vos applications, mais offrent une protection essentielle contre les escalades de privilèges et les compromissions.

---

*Prochain sujet : 11.4 Scan de vulnérabilités - Détecter les failles dans vos images de conteneurs*

⏭️
