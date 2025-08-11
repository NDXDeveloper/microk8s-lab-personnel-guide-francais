🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Configuration DNS locale

## Introduction

La configuration DNS locale est votre première étape vers un environnement de développement professionnel. Au lieu de jongler avec des adresses IP difficiles à mémoriser comme `192.168.1.100:30080`, vous pourrez accéder à vos applications avec des URLs élégantes comme `https://mon-app.lab.local`. Cette section vous guidera à travers la mise en place d'un système DNS local pour votre lab MicroK8s.

## Pourquoi configurer un DNS local ?

### Les bénéfices immédiats

```
Sans DNS local :
━━━━━━━━━━━━━━━━
❌ http://192.168.1.100:30080  → Application web
❌ http://192.168.1.100:30081  → API
❌ http://192.168.1.100:30082  → Dashboard
   Difficile à mémoriser, peu professionnel

Avec DNS local :
━━━━━━━━━━━━━━━━
✅ https://app.lab.local       → Application web
✅ https://api.lab.local       → API
✅ https://dashboard.lab.local → Dashboard
   Simple, mémorable, professionnel
```

### Avantages pour votre lab

1. **Développement réaliste** : Vos applications utilisent des noms de domaine comme en production
2. **Multi-applications** : Hébergez plusieurs apps sur les mêmes ports (80/443)
3. **Certificats SSL** : Même auto-signés, ils fonctionnent mieux avec des noms de domaine
4. **Portabilité** : Vos configurations restent valides même si les IPs changent
5. **Documentation** : Plus facile de documenter avec des noms parlants

## Comprendre la résolution DNS

### Le processus de résolution

Quand vous tapez une URL dans votre navigateur, voici ce qui se passe :

```
Processus de résolution DNS :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Navigateur : "Je veux aller sur app.lab.local"
        ↓
2. Système : "Je consulte mon fichier hosts"
        ↓
3. Fichier hosts : "app.lab.local = 192.168.1.100"
        ↓
4. Navigateur : "OK, je me connecte à 192.168.1.100"
        ↓
5. MicroK8s : "Voici l'application demandée"

📝 Le fichier hosts court-circuite les DNS Internet
```

### Hiérarchie de résolution

Votre système consulte les sources DNS dans cet ordre :

```
Ordre de priorité DNS :
━━━━━━━━━━━━━━━━━━━━━━
1. Cache DNS local (mémoire)
   ↓ (si pas trouvé)
2. Fichier /etc/hosts
   ↓ (si pas trouvé)
3. Serveur DNS configuré (ex: 192.168.1.1)
   ↓ (si pas trouvé)
4. Serveur DNS secondaire
   ↓ (si pas trouvé)
5. Échec de résolution

💡 Notre configuration utilisera l'étape 2
```

## Configuration du fichier hosts

### Localisation du fichier hosts

Le fichier hosts se trouve à différents endroits selon votre système :

```
Emplacement par système :
━━━━━━━━━━━━━━━━━━━━━━━━
• Linux/Mac : /etc/hosts
• Windows   : C:\Windows\System32\drivers\etc\hosts
• WSL2      : /etc/hosts (dans la VM Linux)

⚠️ Nécessite les droits administrateur pour modifier
```

### Structure du fichier hosts

Le fichier hosts suit un format simple :

```
# Format de base
[IP]  [nom.domaine]  [alias optionnel]

# Exemples
127.0.0.1      localhost
192.168.1.100  mon-app.lab.local
192.168.1.100  api.lab.local api

# Commentaires commencent par #
# Une IP peut avoir plusieurs noms
```

### Entrées pour votre lab MicroK8s

Voici les entrées typiques à ajouter pour votre lab :

```bash
# === Configuration DNS Lab MicroK8s ===
# IP de votre machine MicroK8s
192.168.1.100  lab.local

# Applications principales
192.168.1.100  app.lab.local
192.168.1.100  api.lab.local
192.168.1.100  dashboard.lab.local
192.168.1.100  grafana.lab.local
192.168.1.100  prometheus.lab.local

# Environnements de développement
192.168.1.100  dev.lab.local
192.168.1.100  test.lab.local
192.168.1.100  staging.lab.local

# Services techniques
192.168.1.100  registry.lab.local
192.168.1.100  jenkins.lab.local
192.168.1.100  gitlab.lab.local

# Base de données (si exposées)
192.168.1.100  postgres.lab.local
192.168.1.100  mysql.lab.local
192.168.1.100  redis.lab.local
# === Fin Configuration Lab ===
```

### Modification sécurisée du fichier

#### Sur Linux/Mac

```bash
# Faire une sauvegarde d'abord
sudo cp /etc/hosts /etc/hosts.backup

# Éditer avec votre éditeur préféré
sudo nano /etc/hosts
# ou
sudo vim /etc/hosts

# Vérifier les modifications
cat /etc/hosts | grep lab.local
```

#### Sur Windows

```powershell
# Ouvrir PowerShell en administrateur
# Faire une sauvegarde
Copy-Item C:\Windows\System32\drivers\etc\hosts C:\Windows\System32\drivers\etc\hosts.backup

# Éditer avec Notepad en admin
notepad C:\Windows\System32\drivers\etc\hosts

# Ou utiliser PowerShell directement
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "192.168.1.100  app.lab.local"
```

## Choix du domaine local

### Extensions recommandées

```
Domaines pour usage local :
━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Recommandés :
• .local     : Standard pour réseaux locaux
• .lan       : Alternative courante
• .home      : Usage domestique
• .internal  : Réseau interne

⚠️ À éviter :
• .com/.org/.net : Réservés Internet
• .dev        : Propriété de Google, force HTTPS
• .app        : Idem, HSTS activé
• .test       : OK mais réservé pour tests

🏆 Choix optimal : .lab.local
```

### Convention de nommage

Adoptez une convention cohérente pour vos noms :

```
Structure suggérée :
━━━━━━━━━━━━━━━━━━━
[service].[environnement].lab.local

Exemples :
• app.dev.lab.local      → Application en dev
• api.prod.lab.local     → API en production
• dashboard.lab.local    → Service unique

Ou plus simple :
[service].lab.local

Exemples :
• wordpress.lab.local
• nextcloud.lab.local
• gitea.lab.local
```

## Configuration DNS pour plusieurs machines

### Scénario : Accès depuis d'autres appareils

Si vous voulez accéder à votre lab depuis d'autres machines de votre réseau :

```
Configuration multi-machines :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Machine MicroK8s (192.168.1.100)
     ↑
     ├── PC Bureau : Modifier /etc/hosts
     ├── Laptop : Modifier /etc/hosts
     ├── Smartphone : Plus complexe (voir ci-dessous)
     └── Tablette : Plus complexe

💡 Chaque appareil doit connaître les noms DNS
```

### Solutions pour appareils mobiles

Les appareils mobiles ne permettent pas facilement de modifier le fichier hosts :

```
Options pour mobiles/tablettes :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. DNS local sur routeur (optimal)
   → Configure une fois pour tout le réseau

2. App DNS locale (Android: Hosts Go, iOS: complexe)
   → Nécessite souvent root/jailbreak

3. Serveur DNS local (Pi-hole, dnsmasq)
   → Solution robuste mais plus complexe

4. Utiliser les IPs directement
   → Pas idéal mais fonctionne
```

## Mise en place d'un serveur DNS local (optionnel)

### Pourquoi un serveur DNS local ?

```
Avantages d'un serveur DNS :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Configuration centralisée
✅ Tous les appareils du réseau en profitent
✅ Gestion dynamique des entrées
✅ Wildcards possibles (*.lab.local)
✅ Interface web de gestion

❌ Plus complexe à mettre en place
❌ Un service de plus à maintenir
```

### Option 1 : dnsmasq (simple)

Dnsmasq est un serveur DNS léger parfait pour un lab :

```bash
# Installation sur Ubuntu/Debian
sudo apt-get install dnsmasq

# Configuration de base
sudo nano /etc/dnsmasq.conf
```

Configuration minimale dnsmasq :
```conf
# Écouter sur toutes les interfaces
interface=*

# Domaine local
local=/lab.local/

# Entrées DNS
address=/lab.local/192.168.1.100
address=/app.lab.local/192.168.1.100
address=/api.lab.local/192.168.1.100

# Wildcard pour tout *.lab.local
address=/.lab.local/192.168.1.100

# DNS upstream (Google DNS)
server=8.8.8.8
server=8.8.4.4
```

### Option 2 : CoreDNS dans MicroK8s

MicroK8s inclut déjà CoreDNS pour le cluster. Vous pouvez l'étendre :

```yaml
# ConfigMap CoreDNS personnalisé
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  lab.server: |
    lab.local:53 {
      hosts {
        192.168.1.100 app.lab.local
        192.168.1.100 api.lab.local
        192.168.1.100 dashboard.lab.local
        fallthrough
      }
    }
```

### Option 3 : Pi-hole (interface graphique)

Pi-hole offre une interface web conviviale :

```bash
# Installation automatique
curl -sSL https://install.pi-hole.net | bash

# Accès interface : http://pi.hole/admin
# Configuration DNS local dans l'interface
```

## Test de la configuration DNS

### Tests de base

```bash
# Test ping
ping app.lab.local

# Test de résolution DNS
nslookup app.lab.local
# ou
dig app.lab.local

# Test avec curl
curl http://app.lab.local

# Vider le cache DNS si nécessaire
# Linux
sudo systemd-resolve --flush-caches
# Mac
sudo dscacheutil -flushcache
# Windows
ipconfig /flushdns
```

### Test depuis le navigateur

```
Étapes de vérification :
━━━━━━━━━━━━━━━━━━━━━━━━
1. Ouvrir le navigateur
2. Taper : http://app.lab.local
3. Si échec, vérifier :
   • Fichier hosts sauvegardé ?
   • IP correcte ?
   • Service MicroK8s lancé ?
   • Firewall bloque ?
```

### Diagnostiquer les problèmes

```
Problème → Solution
━━━━━━━━━━━━━━━━━━━
"Impossible de résoudre l'hôte"
→ Vérifier le fichier hosts
→ Vider le cache DNS

"Connexion refusée"
→ Service non démarré
→ Mauvais port
→ Firewall bloque

"Certificat invalide"
→ Normal avec DNS local
→ Accepter l'exception

"Fonctionne en IP mais pas en nom"
→ Problème DNS pur
→ Vérifier orthographe
```

## Intégration avec MicroK8s Ingress

### Configuration Ingress pour DNS local

Une fois le DNS local configuré, vos Ingress peuvent utiliser ces noms :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: api.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Wildcard DNS et Ingress

Avec un wildcard DNS (*.lab.local), vous pouvez créer dynamiquement des sous-domaines :

```yaml
# Ingress pour branches Git
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: feature-branch-ingress
spec:
  rules:
  - host: feature-xyz.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-feature-xyz
            port:
              number: 80
```

## Cas d'usage avancés

### Environnements multiples

```
Structure pour multi-environnements :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# Développement
192.168.1.100  dev.app.lab.local
192.168.1.100  dev.api.lab.local

# Test
192.168.1.100  test.app.lab.local
192.168.1.100  test.api.lab.local

# Staging
192.168.1.100  staging.app.lab.local
192.168.1.100  staging.api.lab.local

# Production locale
192.168.1.100  prod.app.lab.local
192.168.1.100  prod.api.lab.local
```

### Simulation de microservices

```
DNS pour architecture microservices :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
192.168.1.100  gateway.lab.local
192.168.1.100  auth.lab.local
192.168.1.100  users.lab.local
192.168.1.100  products.lab.local
192.168.1.100  orders.lab.local
192.168.1.100  payments.lab.local
192.168.1.100  notifications.lab.local
```

## Automatisation de la configuration

### Script de configuration automatique

Créez un script pour automatiser l'ajout des entrées DNS :

```bash
#!/bin/bash
# setup-dns.sh

# Configuration
MICROK8S_IP="192.168.1.100"
DOMAIN="lab.local"
HOSTS_FILE="/etc/hosts"

# Services à configurer
SERVICES=(
    "app"
    "api"
    "dashboard"
    "grafana"
    "prometheus"
    "registry"
)

# Backup
sudo cp $HOSTS_FILE "$HOSTS_FILE.backup.$(date +%Y%m%d)"

# Marker pour nos entrées
echo "# === MicroK8s Lab DNS - START ===" | sudo tee -a $HOSTS_FILE

# Ajouter chaque service
for service in "${SERVICES[@]}"; do
    entry="$MICROK8S_IP  $service.$DOMAIN"
    echo "Ajout: $entry"
    echo "$entry" | sudo tee -a $HOSTS_FILE
done

echo "# === MicroK8s Lab DNS - END ===" | sudo tee -a $HOSTS_FILE

echo "Configuration DNS terminée !"
echo "Test: ping app.$DOMAIN"
```

### Script de nettoyage

```bash
#!/bin/bash
# cleanup-dns.sh

# Supprime nos entrées DNS
sudo sed -i '/=== MicroK8s Lab DNS - START ===/,/=== MicroK8s Lab DNS - END ===/d' /etc/hosts

echo "Entrées DNS du lab supprimées"
```

## Bonnes pratiques

### Organisation du fichier hosts

```
Structure recommandée :
━━━━━━━━━━━━━━━━━━━━━━
# === Système (ne pas toucher) ===
127.0.0.1       localhost
::1             localhost

# === Réseau local ===
192.168.1.1     router.local
192.168.1.10    nas.local

# === Lab MicroK8s ===
192.168.1.100   lab.local
192.168.1.100   app.lab.local
192.168.1.100   api.lab.local

# === Projets temporaires ===
192.168.1.100   test1.lab.local
```

### Documentation

Maintenez une documentation de vos DNS :

```markdown
# DNS Lab MicroK8s

## Machine principale
- IP: 192.168.1.100
- Domaine base: lab.local

## Services déployés
| Service    | URL                    | Port | Description        |
|------------|------------------------|------|--------------------|
| App Web    | app.lab.local         | 443  | Application principale |
| API        | api.lab.local         | 443  | Backend API        |
| Dashboard  | dashboard.lab.local   | 443  | Kubernetes Dashboard |
| Grafana    | grafana.lab.local     | 443  | Monitoring         |
| Prometheus | prometheus.lab.local  | 443  | Métriques          |
```

## Transition vers un domaine public

La configuration DNS locale est parfaite pour le développement, mais vous voudrez éventuellement :

1. **Acquérir un vrai domaine** (section 3.3)
2. **Configurer les DNS publics** (section 3.4)
3. **Obtenir des certificats SSL valides** (section 5)

La bonne nouvelle : toute votre configuration Ingress restera valide, vous changerez juste les noms de domaine !

## Points clés à retenir

1. **Le fichier hosts est votre ami** : Simple, efficace, sans dépendance
2. **Utilisez .lab.local** : Évite les conflits avec les domaines Internet
3. **Documentez vos entrées** : Vous vous remercierez plus tard
4. **Pensez wildcard** : *.lab.local simplifie la gestion
5. **Automatisez** : Scripts pour ajouter/supprimer des entrées

## Résumé

La configuration DNS locale transforme votre lab MicroK8s d'un ensemble d'IPs et de ports en un environnement professionnel avec des URLs mémorables. C'est une étape simple mais qui améliore grandement votre expérience de développement. Dans la prochaine section, nous verrons comment acquérir un vrai nom de domaine pour rendre votre lab accessible depuis Internet.

⏭️
