🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C - Configuration DNS complète

## Introduction au DNS pour Kubernetes

Le DNS (Domain Name System) est le système qui traduit les noms de domaine (comme `app.example.com`) en adresses IP. Dans un environnement Kubernetes, le DNS joue un rôle crucial à deux niveaux : la résolution interne entre services et l'accès externe à vos applications.

### Pourquoi le DNS est Important

Sans configuration DNS appropriée :
- Vos applications ne seront accessibles que par IP
- Les services ne pourront pas se découvrir entre eux
- Les certificats SSL ne fonctionneront pas correctement
- L'expérience utilisateur sera dégradée

### Les Trois Niveaux de DNS

1. **DNS Interne** : CoreDNS dans Kubernetes pour la communication entre pods
2. **DNS Local** : Votre réseau local (home lab)
3. **DNS Public** : Internet (noms de domaine publics)

## Partie 1 : DNS Interne Kubernetes (CoreDNS)

### Comprendre CoreDNS

CoreDNS est le serveur DNS intégré à Kubernetes qui permet :
- La résolution des noms de services
- La découverte automatique des services
- La communication inter-pods par nom

### Vérification et Activation de CoreDNS

```bash
# Vérifier si CoreDNS est activé dans MicroK8s
microk8s status | grep dns

# Si non activé, l'activer
microk8s enable dns

# Vérifier que les pods CoreDNS fonctionnent
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Résultat attendu :
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-7745dcdc5b-xkfqn   1/1     Running   0          5m
```

### Configuration de Base CoreDNS

```bash
# Voir la configuration actuelle
kubectl get configmap coredns -n kube-system -o yaml

# Éditer la configuration si nécessaire
kubectl edit configmap coredns -n kube-system
```

Configuration type de CoreDNS :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        # Forwarders vers DNS externes
        forward . 8.8.8.8 8.8.4.4 {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### Formats de Noms DNS Internes

Dans Kubernetes, chaque service a automatiquement un nom DNS :

```
# Format des noms DNS internes
<service-name>.<namespace>.svc.cluster.local

# Exemples :
mongodb.production.svc.cluster.local    # Service complet
mongodb.production.svc                  # Version courte
mongodb.production                      # Plus court
mongodb                                 # Si même namespace
```

### Test de Résolution DNS Interne

```bash
# Créer un pod de test
kubectl run -it --rm debug --image=busybox --restart=Never -- sh

# Dans le pod, tester la résolution DNS
nslookup kubernetes.default
nslookup kube-dns.kube-system

# Test avec dig (pod avec plus d'outils)
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- sh
dig kubernetes.default.svc.cluster.local
dig @10.152.183.10 google.com  # Test forward externe
```

### Configuration DNS pour Services Headless

Pour les services qui nécessitent une découverte de tous les pods :

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  namespace: production
spec:
  clusterIP: None  # Service headless
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
---
# Résultat DNS :
# mongodb-headless.production.svc.cluster.local retourne toutes les IPs des pods
# mongodb-0.mongodb-headless.production.svc.cluster.local (pour StatefulSet)
# mongodb-1.mongodb-headless.production.svc.cluster.local
```

## Partie 2 : Configuration DNS Local (Home Lab)

### Option A : Utilisation du Fichier Hosts

La méthode la plus simple pour un lab personnel :

#### Sur Windows
```powershell
# Éditer en tant qu'administrateur
notepad C:\Windows\System32\drivers\etc\hosts

# Ajouter vos entrées
192.168.1.100  myapp.local
192.168.1.100  dashboard.local
192.168.1.100  grafana.local
```

#### Sur Linux/Mac
```bash
# Éditer avec sudo
sudo nano /etc/hosts

# Ajouter vos entrées
192.168.1.100  myapp.local
192.168.1.100  dashboard.local
192.168.1.100  grafana.local

# Vérifier
ping myapp.local
```

### Option B : DNS Local avec dnsmasq

Pour une solution plus évoluée :

```bash
# Installation sur Ubuntu/Debian
sudo apt-get update
sudo apt-get install dnsmasq

# Configuration de base
sudo nano /etc/dnsmasq.conf
```

Configuration dnsmasq :
```conf
# /etc/dnsmasq.conf
# Écouter sur l'interface locale
interface=eth0
bind-interfaces

# Domaine local
local=/local/
domain=local

# Entrées DNS statiques
address=/myapp.local/192.168.1.100
address=/dashboard.local/192.168.1.100
address=/grafana.local/192.168.1.100

# Wildcard pour sous-domaines
address=/.apps.local/192.168.1.100

# Forwarder pour le reste
server=8.8.8.8
server=8.8.4.4

# Cache
cache-size=1000
```

```bash
# Redémarrer dnsmasq
sudo systemctl restart dnsmasq

# Tester
nslookup myapp.local 127.0.0.1
```

### Option C : Pi-hole comme DNS Local

Pi-hole offre une interface web et plus de fonctionnalités :

```bash
# Installation automatique
curl -sSL https://install.pi-hole.net | bash

# Ou avec Docker
docker run -d \
  --name pihole \
  -p 53:53/tcp -p 53:53/udp \
  -p 80:80 \
  -e TZ="Europe/Paris" \
  -e WEBPASSWORD="admin" \
  -v pihole:/etc/pihole \
  -v dnsmasq:/etc/dnsmasq.d \
  --restart=unless-stopped \
  pihole/pihole:latest
```

Configuration dans l'interface Pi-hole :
1. Accéder à `http://pihole.local/admin`
2. Local DNS → DNS Records
3. Ajouter vos domaines et IPs

### Configuration des Clients du Réseau Local

#### Configuration Automatique via DHCP

Sur votre routeur :
1. Paramètres DHCP
2. DNS Principal : IP de votre serveur DNS local
3. DNS Secondaire : 8.8.8.8

#### Configuration Manuelle

```bash
# Linux - NetworkManager
sudo nmcli connection modify "Wired connection 1" \
  ipv4.dns "192.168.1.10" \
  ipv4.ignore-auto-dns yes

# Windows PowerShell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.1.10, 8.8.8.8
```

## Partie 3 : Configuration DNS Public

### Acquisition d'un Nom de Domaine

#### Fournisseurs Recommandés pour Labs

1. **Domaines Gratuits** (pour tests) :
   - Freenom (.tk, .ml, .ga)
   - DuckDNS (sous-domaines)
   - No-IP (DNS dynamique)

2. **Domaines Payants** (plus fiables) :
   - Namecheap (~10€/an)
   - Cloudflare Registrar (prix coûtant)
   - OVH (~5-15€/an)

### Configuration avec DuckDNS (Gratuit)

```bash
# 1. Créer un compte sur https://www.duckdns.org

# 2. Créer un sous-domaine (ex: monlab.duckdns.org)

# 3. Script de mise à jour automatique
cat << 'EOF' > ~/update-duckdns.sh
#!/bin/bash
DOMAIN="monlab"
TOKEN="votre-token-duckdns"

# Mise à jour de l'IP
curl -s "https://www.duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip="
echo
EOF

chmod +x ~/update-duckdns.sh

# 4. Automatiser avec cron
crontab -e
# Ajouter :
*/5 * * * * ~/update-duckdns.sh >/dev/null 2>&1
```

### Configuration avec Cloudflare (Recommandé)

Cloudflare offre des services DNS gratuits avec de nombreux avantages :

#### Étape 1 : Configuration Initiale

1. Créer un compte sur [cloudflare.com](https://cloudflare.com)
2. Ajouter votre domaine
3. Changer les nameservers chez votre registrar

#### Étape 2 : Configuration des Enregistrements DNS

```bash
# Via l'interface web Cloudflare ou API

# Enregistrement A pour le domaine principal
Type: A
Name: @
Value: VOTRE_IP_PUBLIQUE
TTL: Auto
Proxy: Off (pour commencer)

# Enregistrement pour wildcard
Type: A
Name: *
Value: VOTRE_IP_PUBLIQUE
TTL: Auto
Proxy: Off

# Enregistrements spécifiques
Type: A
Name: app
Value: VOTRE_IP_PUBLIQUE

Type: A
Name: dashboard
Value: VOTRE_IP_PUBLIQUE
```

#### Étape 3 : Automatisation avec l'API Cloudflare

```bash
# Script de mise à jour DNS dynamique
cat << 'EOF' > ~/cloudflare-ddns.sh
#!/bin/bash

# Configuration
AUTH_EMAIL="votre-email@example.com"
AUTH_KEY="votre-api-key-cloudflare"
ZONE_ID="votre-zone-id"
RECORD_NAME="monlab.example.com"
RECORD_ID="id-de-l-enregistrement"

# Obtenir l'IP publique actuelle
IP=$(curl -s https://ipv4.icanhazip.com)

# Mettre à jour l'enregistrement DNS
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "X-Auth-Email: $AUTH_EMAIL" \
  -H "X-Auth-Key: $AUTH_KEY" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"$RECORD_NAME\",\"content\":\"$IP\",\"ttl\":120,\"proxied\":false}"

echo "DNS mis à jour avec IP: $IP"
EOF

chmod +x ~/cloudflare-ddns.sh

# Automatiser
crontab -e
# */10 * * * * ~/cloudflare-ddns.sh
```

### Configuration avec un Domaine Personnel

Si vous avez acheté un domaine :

```yaml
# Configuration DNS typique chez le registrar
# (Interface web du fournisseur)

# Enregistrements de base
A     @           VOTRE_IP        # Domaine principal
A     www         VOTRE_IP        # www.votredomaine.com
A     *           VOTRE_IP        # Wildcard
CNAME dashboard   @               # dashboard.votredomaine.com
CNAME grafana     @               # grafana.votredomaine.com
CNAME app         @               # app.votredomaine.com

# Pour email (optionnel)
MX    @           mail.votredomaine.com  10
TXT   @           "v=spf1 ip4:VOTRE_IP ~all"
```

## Partie 4 : Configuration de l'Ingress avec DNS

### Configuration Basique de l'Ingress

```yaml
# ingress-with-dns.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  # Règle pour app.example.com
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80

  # Règle pour dashboard.example.com
  - host: dashboard.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dashboard-service
            port:
              number: 80
```

### Configuration avec Wildcard DNS

```yaml
# ingress-wildcard.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
  namespace: apps
spec:
  ingressClassName: nginx
  rules:
  # Wildcard pour *.apps.example.com
  - host: "*.apps.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
---
# Ingress spécifiques pour override
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: specific-app-ingress
  namespace: apps
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.apps.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 8080
```

### Configuration SSL/TLS avec Cert-Manager

```yaml
# cluster-issuer-letsencrypt.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
# ingress-with-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

## Partie 5 : Configuration Réseau et Routeur

### Configuration du Port Forwarding

Pour rendre vos services accessibles depuis Internet :

#### Sur la Box/Routeur

1. Accéder à l'interface admin (généralement `192.168.1.1`)
2. NAT/PAT ou Port Forwarding
3. Ajouter les règles :

```
Port Externe → Port Interne → IP Locale
80          → 80          → 192.168.1.100 (IP MicroK8s)
443         → 443         → 192.168.1.100
6443        → 6443        → 192.168.1.100 (API Kubernetes, optionnel)
```

#### Configuration avec iptables (Linux)

```bash
# Si votre serveur MicroK8s fait office de routeur

# Activer le forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Règles NAT
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80
iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.1.100:443
iptables -t nat -A POSTROUTING -j MASQUERADE

# Sauvegarder les règles
iptables-save > /etc/iptables/rules.v4
```

### Configuration avec IP Fixe

```bash
# Ubuntu avec Netplan
sudo nano /etc/netplan/01-netcfg.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
# Appliquer la configuration
sudo netplan apply
```

### Configuration DHCP Réservation

Sur votre routeur, réserver l'IP pour votre serveur MicroK8s :
1. DHCP → Réservation d'adresse
2. Ajouter l'adresse MAC du serveur
3. Assigner l'IP fixe (ex: 192.168.1.100)

## Partie 6 : DNS Split-Horizon

Pour accéder à vos services avec le même nom depuis l'intérieur et l'extérieur :

### Configuration avec CoreDNS

```yaml
# configmap-coredns-custom.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  example.server: |
    example.com:53 {
        errors
        cache 30
        forward . 192.168.1.10  # Votre DNS local
    }

  internal.server: |
    internal.local:53 {
        errors
        cache 30
        hosts {
          192.168.1.100 app.example.com
          192.168.1.100 dashboard.example.com
          fallthrough
        }
    }
```

```bash
# Appliquer et redémarrer CoreDNS
kubectl apply -f configmap-coredns-custom.yaml
kubectl rollout restart deployment/coredns -n kube-system
```

### Configuration avec MetalLB

Pour avoir des IPs dédiées par service :

```bash
# Activer MetalLB
microk8s enable metallb:192.168.1.200-192.168.1.210

# Service avec IP fixe
```

```yaml
# service-with-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-lb
  annotations:
    metallb.universe.tf/address-pool: default
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.200  # IP fixe dans la plage
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

## Partie 7 : Monitoring et Dépannage DNS

### Outils de Diagnostic DNS

```bash
# Test de résolution basique
nslookup app.example.com
dig app.example.com
host app.example.com

# Test avec serveur DNS spécifique
nslookup app.example.com 8.8.8.8
dig @192.168.1.10 app.example.com

# Trace complète de résolution
dig +trace app.example.com

# Vérifier les enregistrements
dig ANY example.com

# Test depuis un pod Kubernetes
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash
# Dans le pod :
nslookup kubernetes.default
nslookup app-service.default.svc.cluster.local
dig +short app.example.com
```

### Problèmes Courants et Solutions

#### Problème : DNS ne résout pas dans les pods

```bash
# Vérifier CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Redémarrer CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# Vérifier la ConfigMap
kubectl get configmap coredns -n kube-system -o yaml
```

#### Problème : Nom de domaine inaccessible depuis l'extérieur

```bash
# 1. Vérifier l'IP publique
curl ifconfig.me

# 2. Vérifier les enregistrements DNS
nslookup votre-domaine.com 8.8.8.8

# 3. Vérifier le port forwarding
nc -zv VOTRE_IP_PUBLIQUE 80
nc -zv VOTRE_IP_PUBLIQUE 443

# 4. Vérifier l'Ingress
kubectl get ingress
kubectl describe ingress
```

#### Problème : Certificat SSL ne fonctionne pas

```bash
# Vérifier que le DNS pointe bien vers votre IP
dig app.example.com

# Vérifier que l'Ingress est configuré
kubectl get ingress
kubectl describe certificate

# Vérifier les logs de cert-manager
kubectl logs -n cert-manager deployment/cert-manager

# Test manuel ACME challenge
curl http://app.example.com/.well-known/acme-challenge/test
```

### Dashboard de Monitoring DNS

```yaml
# dns-test-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dns-monitor
  namespace: monitoring
spec:
  schedule: "*/5 * * * *"  # Toutes les 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: dns-test
            image: nicolaka/netshoot
            command:
            - /bin/bash
            - -c
            - |
              echo "=== DNS Test $(date) ==="

              # Test DNS interne
              nslookup kubernetes.default || echo "FAIL: kubernetes.default"

              # Test DNS externe
              nslookup google.com || echo "FAIL: google.com"

              # Test vos domaines
              nslookup app.example.com || echo "FAIL: app.example.com"

              # Envoyer métriques à Prometheus (optionnel)
              # curl -X POST http://pushgateway:9091/metrics/job/dns-test
          restartPolicy: OnFailure
```

## Partie 8 : Configurations Avancées

### Configuration Multi-Domaines

```yaml
# ingress-multi-domain.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-domain-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.domain1.com
    secretName: domain1-tls
  - hosts:
    - app.domain2.com
    secretName: domain2-tls
  rules:
  - host: app.domain1.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1
            port:
              number: 80
  - host: app.domain2.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2
            port:
              number: 80
```

### Configuration ExternalDNS

Pour automatiser la création d'enregistrements DNS :

```yaml
# external-dns-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.12.0
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=example.com
        - --provider=cloudflare
        - --cloudflare-proxied=false
        env:
        - name: CF_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: cloudflare-api-token
              key: token
---
# Avec cette configuration, les Ingress créent automatiquement les DNS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-dns-ingress
  annotations:
    external-dns.alpha.kubernetes.io/hostname: auto.example.com
spec:
  rules:
  - host: auto.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port:
              number: 80
```

### Configuration DNS over HTTPS (DoH)

```yaml
# coredns-doh.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        # DNS over HTTPS
        forward . tls://1.1.1.1 tls://1.0.0.1 {
            tls_servername cloudflare-dns.com
            health_check 5s
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## Scripts Utiles

### Script de Test DNS Complet

```bash
#!/bin/bash
# dns-test-complete.sh

echo "==================================="
echo "Test DNS Complet - $(date)"
echo "==================================="

# Couleurs
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

# Fonction de test
test_dns() {
    local domain=$1
    local expected_ip=$2

    result=$(dig +short $domain)

    if [ "$result" = "$expected_ip" ]; then
        echo -e "${GREEN}✓${NC} $domain → $result"
    else
        echo -e "${RED}✗${NC} $domain → $result (attendu: $expected_ip)"
    fi
}

echo -e "\n1. Test DNS Interne Kubernetes"
kubectl run -it --rm dns-test --image=busybox --restart=Never -- nslookup kubernetes.default

echo -e "\n2. Test DNS Local"
test_dns "myapp.local" "192.168.1.100"
test_dns "dashboard.local" "192.168.1.100"

echo -e "\n3. Test DNS Public"
test_dns "example.com" "$(curl -s ifconfig.me)"

echo -e "\n4. Test Résolution Inverse"
dig -x 192.168.1.100

echo -e "\n5. Test Propagation DNS"
echo "Cloudflare:" $(dig @1.1.1.1 +short example.com)
echo "Google:" $(dig @8.8.8.8 +short example.com)
echo "Quad9:" $(dig @9.9.9.9 +short example.com)

echo -e "\n==================================="
```

### Script de Configuration Automatique DNS

```bash
#!/bin/bash
# setup-dns-auto.sh

DOMAIN=${1:-example.com}
IP=${2:-$(curl -s ifconfig.me)}

echo "Configuration DNS pour $DOMAIN → $IP"

# 1. Mise à jour hosts local
echo "$IP $DOMAIN" | sudo tee -a /etc/hosts
echo "$IP *.$DOMAIN" | sudo tee -a /etc/hosts

# 2. Configuration Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
  - host: "*.$DOMAIN"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
EOF

echo "Configuration terminée !"
```

## Checklist de Configuration DNS

### ✅ Configuration Initiale

- [ ] CoreDNS activé dans MicroK8s
- [ ] CoreDNS pods en Running
- [ ] Test de résolution interne fonctionnel
- [ ] Forward vers DNS externe configuré

### ✅ Configuration Locale

- [ ] Entrées hosts ou DNS local configurées
- [ ] Clients réseau configurés pour utiliser le DNS local
- [ ] Test de résolution locale réussi
- [ ] Wildcard configuré si nécessaire

### ✅ Configuration Publique

- [ ] Nom de domaine acquis
- [ ] Enregistrements DNS configurés
- [ ] TTL approprié (120-300s pour tests)
- [ ] Propagation DNS vérifiée

### ✅ Configuration Réseau

- [ ] IP fixe ou réservation DHCP
- [ ] Port forwarding 80/443 configuré
- [ ] Firewall autorise le trafic
- [ ] Test depuis l'extérieur réussi

### ✅ Configuration Ingress

- [ ] Ingress Controller actif
- [ ] Règles Ingress créées
- [ ] Hosts corrects dans Ingress
- [ ] Certificats SSL configurés

### ✅ Tests

- [ ] Résolution DNS interne
- [ ] Résolution DNS externe
- [ ] Accès local aux services
- [ ] Accès externe aux services
- [ ] HTTPS fonctionnel

## Dépannage DNS Rapide

### Arbre de Décision DNS

```
Le DNS fonctionne-t-il ?
├─ Test: nslookup kubernetes.default (dans un pod)
├─ NON → CoreDNS problem
│   ├─ kubectl get pods -n kube-system | grep dns
│   ├─ kubectl logs -n kube-system -l k8s-app=kube-dns
│   └─ kubectl rollout restart deployment/coredns -n kube-system
│
├─ OUI → DNS interne OK
│   ├─ Test: nslookup google.com (dans un pod)
│   ├─ NON → Forward DNS problem
│   │   ├─ Vérifier ConfigMap coredns
│   │   └─ Vérifier forward servers
│   │
│   └─ OUI → DNS externe OK
│       ├─ Test: nslookup votre-domaine.com
│       ├─ NON → Configuration domaine
│       │   ├─ Vérifier enregistrements DNS
│       │   ├─ Vérifier propagation
│       │   └─ Vérifier TTL
│       │
│       └─ OUI → Tout fonctionne !
```

### Solutions aux Problèmes Fréquents

#### "Name or service not known"

```bash
# 1. Vérifier que CoreDNS fonctionne
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Tester avec IP directe
kubectl exec POD_NAME -- nc -zv 10.152.183.10 53

# 3. Vérifier /etc/resolv.conf dans le pod
kubectl exec POD_NAME -- cat /etc/resolv.conf
# Doit contenir :
# nameserver 10.152.183.10
# search default.svc.cluster.local svc.cluster.local cluster.local

# 4. Si incorrect, vérifier dnsPolicy du pod
kubectl get pod POD_NAME -o yaml | grep dnsPolicy
# Doit être : dnsPolicy: ClusterFirst
```

#### "Connection timed out"

```bash
# 1. Vérifier la connectivité réseau
kubectl exec POD_NAME -- ping 8.8.8.8

# 2. Vérifier les NetworkPolicies
kubectl get networkpolicies --all-namespaces

# 3. Vérifier le firewall du node
sudo iptables -L -n | grep 53

# 4. Tester avec un autre DNS
kubectl exec POD_NAME -- nslookup google.com 8.8.8.8
```

#### "NXDOMAIN"

```bash
# 1. Vérifier l'orthographe du domaine
echo "Domaine testé : $DOMAIN"

# 2. Vérifier la propagation DNS
dig @8.8.8.8 $DOMAIN
dig @1.1.1.1 $DOMAIN

# 3. Vérifier les enregistrements chez le registrar
# Interface web du fournisseur DNS

# 4. Attendre la propagation (jusqu'à 48h pour certains)
watch -n 60 "dig +short $DOMAIN"
```

#### "No route to host"

```bash
# 1. Vérifier l'IP du service
kubectl get svc
kubectl get endpoints

# 2. Vérifier que les pods backend existent
kubectl get pods -l app=YOUR_APP

# 3. Tester la connectivité directe
kubectl exec POD_NAME -- curl http://SERVICE_IP:PORT

# 4. Vérifier les labels et selectors
kubectl describe svc SERVICE_NAME
kubectl get pods --show-labels
```

## Exemples de Configurations Complètes

### Configuration 1 : Lab Simple avec Hosts Local

```bash
# 1. Configuration /etc/hosts sur votre machine
sudo bash -c 'cat >> /etc/hosts << EOF
192.168.1.100 k8s.local
192.168.1.100 app.k8s.local
192.168.1.100 dashboard.k8s.local
192.168.1.100 grafana.k8s.local
EOF'

# 2. Déploiement des services
kubectl apply -f - <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: app.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
EOF

# 3. Test
curl http://app.k8s.local
```

### Configuration 2 : Production avec Domaine Public

```bash
# 1. Configuration DNS Cloudflare (via API)
curl -X POST "https://api.cloudflare.com/client/v4/zones/ZONE_ID/dns_records" \
  -H "X-Auth-Email: email@example.com" \
  -H "X-Auth-Key: API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "type":"A",
    "name":"k8s",
    "content":"YOUR_PUBLIC_IP",
    "ttl":120,
    "proxied":false
  }'

# 2. Configuration avec SSL automatique
kubectl apply -f - <<EOF
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - k8s.example.com
    secretName: k8s-tls
  rules:
  - host: k8s.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
EOF

# 3. Vérification SSL
curl https://k8s.example.com
openssl s_client -connect k8s.example.com:443 -servername k8s.example.com
```

### Configuration 3 : Multi-Tenant avec Sous-Domaines

```yaml
# multi-tenant-dns.yaml
---
# Namespace par tenant
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
---
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-b
---
# Ingress pour tenant A
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-a-ingress
  namespace: tenant-a
spec:
  rules:
  - host: tenant-a.k8s.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: "*.tenant-a.k8s.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
---
# Ingress pour tenant B
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tenant-b-ingress
  namespace: tenant-b
spec:
  rules:
  - host: tenant-b.k8s.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

## Automatisation avec Scripts

### Script de Configuration DNS Complète

```bash
#!/bin/bash
# setup-complete-dns.sh

set -e

# Configuration
DOMAIN="${1:-example.local}"
PUBLIC_IP="${2:-$(curl -s ifconfig.me)}"
LOCAL_IP="${3:-192.168.1.100}"
DNS_PROVIDER="${4:-local}"  # local, cloudflare, route53

echo "======================================"
echo "Configuration DNS Complète"
echo "Domain: $DOMAIN"
echo "Public IP: $PUBLIC_IP"
echo "Local IP: $LOCAL_IP"
echo "Provider: $DNS_PROVIDER"
echo "======================================"

# Fonction pour configurer le DNS local
setup_local_dns() {
    echo "Configuration DNS local..."

    # Ajouter au fichier hosts
    echo "$LOCAL_IP $DOMAIN" | sudo tee -a /etc/hosts
    echo "$LOCAL_IP *.$DOMAIN" | sudo tee -a /etc/hosts

    # Si dnsmasq est installé
    if command -v dnsmasq &> /dev/null; then
        echo "address=/$DOMAIN/$LOCAL_IP" | sudo tee -a /etc/dnsmasq.conf
        sudo systemctl restart dnsmasq
    fi
}

# Fonction pour configurer Cloudflare
setup_cloudflare_dns() {
    echo "Configuration Cloudflare DNS..."

    # Nécessite les variables d'environnement
    # CF_EMAIL, CF_API_KEY, CF_ZONE_ID

    curl -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records" \
        -H "X-Auth-Email: $CF_EMAIL" \
        -H "X-Auth-Key: $CF_API_KEY" \
        -H "Content-Type: application/json" \
        --data "{
            \"type\":\"A\",
            \"name\":\"$DOMAIN\",
            \"content\":\"$PUBLIC_IP\",
            \"ttl\":120,
            \"proxied\":false
        }"

    # Wildcard
    curl -X POST "https://api.cloudflare.com/client/v4/zones/$CF_ZONE_ID/dns_records" \
        -H "X-Auth-Email: $CF_EMAIL" \
        -H "X-Auth-Key: $CF_API_KEY" \
        -H "Content-Type: application/json" \
        --data "{
            \"type\":\"A\",
            \"name\":\"*.$DOMAIN\",
            \"content\":\"$PUBLIC_IP\",
            \"ttl\":120,
            \"proxied\":false
        }"
}

# Configuration CoreDNS
setup_coredns() {
    echo "Configuration CoreDNS..."

    kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    $DOMAIN:53 {
        errors
        cache 30
        forward . $LOCAL_IP
        reload
    }
EOF

    kubectl rollout restart deployment/coredns -n kube-system
}

# Configuration Ingress
setup_ingress() {
    echo "Configuration Ingress..."

    kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: $DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
  - host: "*.$DOMAIN"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
EOF
}

# Tests
test_dns() {
    echo "Tests DNS..."

    # Test local
    echo -n "Test local: "
    nslookup $DOMAIN 127.0.0.1 > /dev/null 2>&1 && echo "✓" || echo "✗"

    # Test externe
    echo -n "Test externe: "
    nslookup $DOMAIN 8.8.8.8 > /dev/null 2>&1 && echo "✓" || echo "✗"

    # Test Kubernetes
    echo -n "Test Kubernetes: "
    kubectl run -it --rm dns-test --image=busybox --restart=Never -- \
        nslookup kubernetes.default > /dev/null 2>&1 && echo "✓" || echo "✗"
}

# Exécution principale
main() {
    case $DNS_PROVIDER in
        local)
            setup_local_dns
            ;;
        cloudflare)
            setup_cloudflare_dns
            ;;
        *)
            echo "Provider non supporté: $DNS_PROVIDER"
            exit 1
            ;;
    esac

    setup_coredns
    setup_ingress

    echo "Attente de propagation (30s)..."
    sleep 30

    test_dns

    echo "======================================"
    echo "Configuration terminée !"
    echo "Testez avec : curl http://$DOMAIN"
    echo "======================================"
}

main
```

### Script de Monitoring DNS

```bash
#!/bin/bash
# monitor-dns.sh

# Configuration
DOMAINS=("app.example.com" "api.example.com" "dashboard.example.com")
EXPECTED_IP="192.168.1.100"
SLACK_WEBHOOK=""  # Optionnel

# Fonction d'alerte
alert() {
    echo "[$(date)] ALERTE: $1"

    # Envoyer sur Slack si configuré
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"DNS Alert: $1\"}" \
            $SLACK_WEBHOOK
    fi
}

# Test en boucle
while true; do
    for domain in "${DOMAINS[@]}"; do
        result=$(dig +short $domain)

        if [ "$result" != "$EXPECTED_IP" ]; then
            alert "$domain résout vers $result au lieu de $EXPECTED_IP"
        fi

        # Test HTTPS
        if ! curl -s -o /dev/null -w "%{http_code}" https://$domain | grep -q "200"; then
            alert "$domain n'est pas accessible en HTTPS"
        fi
    done

    # Test DNS interne Kubernetes
    if ! kubectl run -it --rm dns-test --image=busybox --restart=Never -- \
        nslookup kubernetes.default > /dev/null 2>&1; then
        alert "DNS interne Kubernetes ne fonctionne pas"
    fi

    sleep 300  # Test toutes les 5 minutes
done
```

## Best Practices DNS

### 1. Sécurité

- **DNSSEC** : Activer si possible pour éviter le DNS spoofing
- **DNS over HTTPS/TLS** : Chiffrer les requêtes DNS
- **Validation** : Toujours valider les entrées DNS
- **Rate Limiting** : Limiter les requêtes pour éviter les abus

### 2. Performance

- **TTL approprié** :
  - Développement : 60-120 secondes
  - Production : 3600 secondes ou plus
- **Cache DNS** : Utiliser le cache CoreDNS
- **Géo-DNS** : Pour les déploiements multi-régions

### 3. Fiabilité

- **DNS secondaire** : Toujours avoir un backup
- **Monitoring** : Surveiller la résolution DNS
- **Documentation** : Documenter toutes les entrées DNS
- **Tests** : Tester avant de modifier en production

### 4. Organisation

```bash
# Structure recommandée pour les noms DNS
example.com                    # Site principal
api.example.com                # API
dashboard.example.com          # Dashboard admin
grafana.example.com           # Monitoring
*.apps.example.com            # Applications utilisateur
*.dev.example.com             # Environnement de dev
*.staging.example.com         # Environnement de staging
```

## Ressources et Références

### Documentation Officielle

- [CoreDNS Documentation](https://coredns.io/manual/toc/)
- [Kubernetes DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Cert-Manager DNS01](https://cert-manager.io/docs/configuration/acme/dns01/)
- [External-DNS](https://github.com/kubernetes-sigs/external-dns)

### Outils Utiles

- **dig** : Outil de diagnostic DNS complet
- **nslookup** : Test de résolution simple
- **host** : Alternative à nslookup
- **dnsperf** : Test de performance DNS
- **dnstop** : Monitoring temps réel DNS

### Services DNS Gratuits

1. **Cloudflare** : DNS rapide avec API
2. **DuckDNS** : DNS dynamique gratuit
3. **No-IP** : Alternative à DuckDNS
4. **Freenom** : Domaines gratuits (.tk, .ml)
5. **Let's Encrypt** : Certificats SSL gratuits

## Conclusion

La configuration DNS est un élément fondamental de votre infrastructure Kubernetes. Une bonne configuration DNS vous permet :

- **Accessibilité** : Services accessibles par des noms mémorables
- **Flexibilité** : Changement d'IP sans impact utilisateur
- **Sécurité** : HTTPS avec certificats valides
- **Scalabilité** : Ajout facile de nouveaux services

Points clés à retenir :

1. **Commencez simple** : Fichier hosts pour les tests
2. **Évoluez progressivement** : DNS local puis public
3. **Automatisez** : Scripts pour les tâches répétitives
4. **Monitorez** : Surveillez la résolution DNS
5. **Documentez** : Gardez trace de vos configurations

Avec cette annexe, vous disposez de toutes les informations nécessaires pour configurer un DNS complet et fonctionnel pour votre lab MicroK8s, depuis la configuration la plus simple jusqu'aux déploiements multi-domaines avec SSL automatique.

---

*Note : Les configurations DNS peuvent varier selon votre environnement réseau et votre fournisseur de domaine. Adaptez les exemples à votre situation spécifique et testez toujours en environnement de développement avant la production.*

⏭️
