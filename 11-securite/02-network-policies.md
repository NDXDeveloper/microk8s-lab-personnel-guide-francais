🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 Network Policies

## Comprendre les Network Policies

Imaginez votre cluster Kubernetes comme un grand bureau open-space où tous les employés (pods) peuvent naturellement se parler entre eux. Les Network Policies sont comme des cloisons et des portes que vous installez pour créer des espaces privés et contrôler qui peut communiquer avec qui. Sans Network Policies, c'est la politique de la porte ouverte : tout pod peut parler à n'importe quel autre pod.

### Le modèle réseau par défaut de Kubernetes

Par défaut, Kubernetes adopte une approche très ouverte :

- **Tous les pods peuvent communiquer entre eux** sans restriction
- **Tous les pods peuvent être joints** depuis n'importe quel namespace
- **Le trafic circule librement** dans toutes les directions

C'est pratique pour le développement, mais dangereux en production. Un pod compromis pourrait scanner et attaquer tous les autres services du cluster.

### Pourquoi les Network Policies sont cruciales

Les Network Policies vous permettent de :

- **Isoler les environnements** : Séparer production, staging et développement
- **Implémenter le principe Zero Trust** : Aucune communication par défaut
- **Limiter les mouvements latéraux** : Un pod compromis ne peut pas tout attaquer
- **Respecter la conformité** : Isoler les données sensibles (PCI-DSS, GDPR)
- **Micro-segmentation** : Chaque application dans sa bulle de sécurité

## Comment fonctionnent les Network Policies

### Le principe de base

Les Network Policies fonctionnent comme des règles de firewall au niveau des pods :

1. **Par défaut, tout est autorisé** (sans policies)
2. **Dès qu'une policy existe**, elle devient restrictive pour les pods qu'elle sélectionne
3. **Les policies sont additives** : plusieurs policies peuvent s'appliquer à un même pod
4. **Elles contrôlent le trafic entrant (Ingress) et sortant (Egress)**

### Anatomie d'une Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ma-policy
  namespace: mon-namespace
spec:
  podSelector:        # Quels pods sont concernés
    matchLabels:
      app: mon-app
  policyTypes:        # Types de trafic à contrôler
  - Ingress
  - Egress
  ingress:            # Règles pour le trafic entrant
  - from:
    - podSelector:
        matchLabels:
          role: frontend
  egress:             # Règles pour le trafic sortant
  - to:
    - podSelector:
        matchLabels:
          role: database
```

### Les trois dimensions du contrôle

**1. Sélection des pods (podSelector)**
- Détermine quels pods sont protégés par cette policy
- Utilise les labels Kubernetes

**2. Direction du trafic (policyTypes)**
- **Ingress** : Contrôle le trafic entrant
- **Egress** : Contrôle le trafic sortant
- Peut avoir l'un, l'autre, ou les deux

**3. Sources/Destinations autorisées**
- **podSelector** : Autres pods dans le cluster
- **namespaceSelector** : Pods d'autres namespaces
- **ipBlock** : Adresses IP externes

## Network Policies dans MicroK8s

### Prérequis : CNI compatible

MicroK8s utilise par défaut Calico comme plugin réseau (CNI), qui supporte les Network Policies :

```bash
# Vérifier que Calico est installé
microk8s kubectl get pods -n kube-system | grep calico

# Si Calico n'est pas présent, l'installer
microk8s enable dns storage ingress metallb:10.64.140.43-10.64.140.49

# Vérifier le support des Network Policies
microk8s kubectl api-resources | grep networkpolicies
```

### Test de connectivité de base

Avant d'appliquer des policies, vérifions la connectivité par défaut :

```bash
# Créer deux pods de test
microk8s kubectl run pod-client --image=busybox --command -- sleep 3600
microk8s kubectl run pod-server --image=nginx

# Obtenir l'IP du pod serveur
microk8s kubectl get pod pod-server -o wide

# Tester la connexion (remplacer par l'IP réelle)
microk8s kubectl exec pod-client -- wget -O- <IP_POD_SERVER>
```

## Types de Network Policies

### 1. Policy de refus total (Deny All)

La base de la sécurité Zero Trust : bloquer tout par défaut.

#### Deny All Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}  # Sélectionne TOUS les pods du namespace
  policyTypes:
  - Ingress        # Bloque tout trafic entrant
```

#### Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress        # Bloque tout trafic sortant
```

#### Deny All (Ingress + Egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 2. Policy d'autorisation sélective

Après avoir tout bloqué, on autorise spécifiquement ce qui est nécessaire.

#### Autoriser depuis un label spécifique

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### Autoriser depuis un namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: mon-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090  # Port des métriques Prometheus
```

### 3. Policy pour trafic externe

#### Autoriser le trafic Internet sortant

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      needs-internet: "true"
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8      # Bloquer le réseau interne
        - 192.168.0.0/16
        - 172.16.0.0/12
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53            # DNS
```

#### Autoriser depuis un load balancer

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-loadbalancer
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/24  # Subnet du load balancer
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
```

## Patterns courants de Network Policies

### Pattern 1 : Architecture 3-tiers

Un pattern classique avec frontend, backend et base de données :

```yaml
# 1. Deny all par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: app-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 2. Frontend peut recevoir du trafic externe
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: app-namespace
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0  # Depuis Internet
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# 3. Backend peut recevoir du frontend et parler à la DB
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: app-namespace
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# 4. Database ne reçoit que du backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: app-namespace
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

### Pattern 2 : Microservices avec service mesh

Pour une architecture microservices :

```yaml
# Politique pour un microservice
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: microservice-a-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: service-a
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway  # Depuis l'API Gateway
    - podSelector:
        matchLabels:
          app: service-b    # Depuis le service B
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: service-c    # Vers le service C
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - podSelector:
        matchLabels:
          app: service-d    # Vers le service D
    ports:
    - protocol: TCP
      port: 9090
  - to:  # DNS toujours nécessaire
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

### Pattern 3 : Isolation par environnement

Séparer dev, staging et production :

```yaml
# Label les namespaces d'abord
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
---
# Policy : Dev ne peut pas parler à Production
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-production
  namespace: production
spec:
  podSelector: {}  # Tous les pods de production
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: production  # Seulement depuis production
    - namespaceSelector:
        matchLabels:
          environment: staging     # Et staging (pour les tests)
  # Pas de dev !
```

## DNS et Network Policies

### Le piège du DNS

Une erreur courante est d'oublier d'autoriser le DNS. Sans DNS, vos pods ne peuvent pas résoudre les noms :

```yaml
# PROBLÈME : Cette policy bloque aussi le DNS !
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrictive-policy-broken
spec:
  podSelector:
    matchLabels:
      app: mon-app
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
```

### Solution : Toujours autoriser le DNS

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrictive-policy-fixed
spec:
  podSelector:
    matchLabels:
      app: mon-app
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # IMPORTANT : Autoriser le DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Combinaison de sélecteurs

### AND vs OR dans les sélecteurs

Les sélecteurs peuvent être combinés de différentes manières :

#### Logique AND (les deux conditions)

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: prod
    podSelector:          # AND - doit être dans prod ET avoir le label
      matchLabels:
        role: frontend
```

#### Logique OR (l'une ou l'autre)

```yaml
ingress:
- from:
  - namespaceSelector:    # OR - depuis le namespace prod
      matchLabels:
        environment: prod
  - podSelector:          # OU depuis n'importe quel pod frontend
      matchLabels:
        role: frontend
```

### Multiples ports et protocoles

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-policy
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: monitor
    ports:
    - protocol: TCP
      port: 9090      # Métriques Prometheus
    - protocol: TCP
      port: 8080      # Health checks
  - from:
    - podSelector:
        matchLabels:
          role: user
    ports:
    - protocol: TCP
      port: 80        # HTTP
    - protocol: TCP
      port: 443       # HTTPS
```

## Test et validation des Network Policies

### Outils de test

#### 1. Création d'un environnement de test

```yaml
# Namespace de test
apiVersion: v1
kind: Namespace
metadata:
  name: netpol-test
---
# Pod client pour tester
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: netpol-test
  labels:
    role: client
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "while true; do sleep 30; done;"]
---
# Pod serveur cible
apiVersion: v1
kind: Pod
metadata:
  name: test-server
  namespace: netpol-test
  labels:
    role: server
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

#### 2. Commandes de test de connectivité

```bash
# Obtenir l'IP du serveur
SERVER_IP=$(microk8s kubectl get pod test-server -n netpol-test -o jsonpath='{.status.podIP}')

# Test depuis le client
microk8s kubectl exec -n netpol-test test-client -- curl -s --max-time 5 http://$SERVER_IP

# Test avec netcat
microk8s kubectl exec -n netpol-test test-client -- nc -zv $SERVER_IP 80

# Test DNS
microk8s kubectl exec -n netpol-test test-client -- nslookup kubernetes.default

# Trace route pour debug
microk8s kubectl exec -n netpol-test test-client -- traceroute $SERVER_IP
```

### Visualisation des Network Policies

```bash
# Lister toutes les policies
microk8s kubectl get networkpolicies --all-namespaces

# Détails d'une policy
microk8s kubectl describe networkpolicy ma-policy -n mon-namespace

# Export en YAML pour analyse
microk8s kubectl get networkpolicy ma-policy -n mon-namespace -o yaml

# Voir quels pods sont affectés par une policy
microk8s kubectl get pods -n mon-namespace -l app=mon-app
```

### Debugging des problèmes

#### Problème 1 : Connexion timeout

```bash
# Vérifier qu'une policy existe
microk8s kubectl get netpol -n mon-namespace

# Vérifier les labels des pods
microk8s kubectl get pods -n mon-namespace --show-labels

# Vérifier les logs Calico
microk8s kubectl logs -n kube-system -l k8s-app=calico-node
```

#### Problème 2 : DNS ne fonctionne pas

```bash
# Test DNS direct
microk8s kubectl exec mon-pod -- nslookup kubernetes.default

# Vérifier la policy DNS
microk8s kubectl get netpol -n mon-namespace -o yaml | grep -A10 "port: 53"

# Test avec l'IP directe du DNS
microk8s kubectl exec mon-pod -- nslookup kubernetes.default 10.152.183.10
```

## Monitoring et audit des Network Policies

### Logs Calico

MicroK8s avec Calico peut logger les connexions bloquées :

```bash
# Activer le logging dans Calico
microk8s kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"logSeverityScreen":"Info","iptablesLogLevel":"7"}}'

# Voir les logs
microk8s kubectl logs -n kube-system -l k8s-app=calico-node --tail=100
```

### Métriques Prometheus

Si vous avez Prometheus installé :

```yaml
# ServiceMonitor pour Calico
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: calico-metrics
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  endpoints:
  - port: metrics
    interval: 30s
```

## Bonnes pratiques

### 1. Commencer par tout bloquer

```yaml
# Appliquer en premier dans chaque namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 2. Labellisation cohérente

```yaml
# Utilisez des labels standards
metadata:
  labels:
    app: mon-app          # Application
    tier: backend         # Niveau architectural
    environment: prod     # Environnement
    team: alpha          # Équipe propriétaire
    version: v1.2.3      # Version
```

### 3. Documentation des policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  annotations:
    description: "Autorise le trafic du frontend vers le backend sur le port 8080"
    owner: "equipe-backend@monlab.com"
    jira: "SEC-1234"
    last-review: "2024-01-15"
spec:
  # ...
```

### 4. Policies par couches

Structure recommandée :

```
1. Global (namespace) : default-deny-all
2. Par tier : frontend-policy, backend-policy, database-policy
3. Par application : app-specific-policy
4. Exceptions : temporary-debug-access
```

### 5. Gestion des namespaces système

```yaml
# N'oubliez pas les namespaces système
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-system-critical
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  # Metrics server
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    podSelector:
      matchLabels:
        k8s-app: metrics-server
```

## Exemples complets

### Exemple 1 : Blog WordPress sécurisé

```yaml
# 1. Namespace avec labels
apiVersion: v1
kind: Namespace
metadata:
  name: wordpress
  labels:
    name: wordpress
---
# 2. Deny all par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: wordpress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 3. WordPress frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: wordpress-netpol
  namespace: wordpress
spec:
  podSelector:
    matchLabels:
      app: wordpress
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx  # Depuis l'ingress controller
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: mysql          # Vers MySQL
    ports:
    - protocol: TCP
      port: 3306
  - to:                       # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to:                       # Updates WordPress
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
---
# 4. MySQL database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-netpol
  namespace: wordpress
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: wordpress
    ports:
    - protocol: TCP
      port: 3306
```

### Exemple 2 : API microservices avec monitoring

```yaml
# API Gateway
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0      # Internet
    ports:
    - protocol: TCP
      port: 443
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring    # Prometheus
    ports:
    - protocol: TCP
      port: 9090
  egress:
  - to:
    - podSelector: {}        # Tous les microservices du namespace
    ports:
    - protocol: TCP
      port: 8080
  - to:                      # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# Microservice générique
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: microservice-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      tier: microservice
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway    # Depuis API Gateway
    ports:
    - protocol: TCP
      port: 8080
  - from:
    - podSelector:
        matchLabels:
          tier: microservice  # Entre microservices
    ports:
    - protocol: TCP
      port: 8080
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring    # Métriques
    ports:
    - protocol: TCP
      port: 9090
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: microservice  # Vers autres microservices
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - podSelector:
        matchLabels:
          app: redis         # Cache
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - podSelector:
        matchLabels:
          app: postgres      # Database
    ports:
    - protocol: TCP
      port: 5432
  - to:                      # DNS
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

## Dépannage avancé

### Checklist de dépannage

1. **La policy existe-t-elle ?**
   ```bash
   microk8s kubectl get netpol -n <namespace>
   ```

2. **Les labels correspondent-ils ?**
   ```bash
   microk8s kubectl get pods -n <namespace> --show-labels
   ```

3. **La policy sélectionne-t-elle les bons pods ?**
   ```bash
   microk8s kubectl describe netpol <policy-name> -n <namespace>
   ```

4. **Le DNS est-il autorisé ?**
   ```bash
   microk8s kubectl exec <pod> -- nslookup kubernetes.default
   ```

5. **Y a-t-il plusieurs policies conflictuelles ?**
   ```bash
   microk8s kubectl get netpol -n <namespace> -o yaml | grep -A5 podSelector
   ```

### Outils de diagnostic

```bash
# Script de test de connectivité
cat << 'EOF' > test-connectivity.sh
#!/bin/bash
NAMESPACE=$1
SOURCE_POD=$2
TARGET_POD=$3
PORT=$4

echo "Testing connectivity from $SOURCE_POD to $TARGET_POD on port $PORT"

# Get target IP
TARGET_IP=$(kubectl get pod $TARGET_POD -n $NAMESPACE -o jsonpath='{.status.podIP}')
echo "Target IP: $TARGET_IP"

# Test connectivity
kubectl exec -n $NAMESPACE $SOURCE_POD -- nc -zv $TARGET_IP $PORT -w 5

# Test DNS
echo "Testing DNS resolution..."
kubectl exec -n $NAMESPACE $SOURCE_POD -- nslookup $TARGET_POD.$NAMESPACE.svc.cluster.local
EOF

chmod +x test-connectivity.sh
```

## Résumé et points clés

Les Network Policies sont votre firewall interne dans Kubernetes :

1. **Par défaut, tout est ouvert** - Kubernetes permet toute communication
2. **Adoptez Zero Trust** - Commencez par tout bloquer, puis ouvrez au besoin
3. **N'oubliez jamais le DNS** - Source d'erreur numéro 1
4. **Les policies sont additives** - Plusieurs policies = union des permissions
5. **Testez systématiquement** - Avant et après chaque changement
6. **Documentez vos décisions** - Les policies complexes nécessitent des explications
7. **Surveillez les logs** - Pour détecter les blocages non voulus

Les Network Policies demandent de la rigueur mais offrent une sécurité essentielle. Commencez simple avec un "deny all", puis construisez progressivement vos règles en testant à chaque étape.

---

*Prochain sujet : 11.3 Pod Security Standards - Définir les contraintes de sécurité au niveau des pods*

⏭️
