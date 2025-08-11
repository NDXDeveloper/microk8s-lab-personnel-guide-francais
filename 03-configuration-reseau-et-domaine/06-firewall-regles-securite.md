🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 Firewall et règles de sécurité

## Introduction

Le firewall est votre gardien numérique. Après avoir ouvert les portes de votre lab au monde avec la redirection de ports, il est crucial de contrôler précisément qui peut entrer et accéder à quoi. Cette section vous guidera à travers la configuration d'un firewall robuste pour protéger votre lab MicroK8s tout en maintenant l'accessibilité nécessaire.

## Comprendre le firewall

### Qu'est-ce qu'un firewall ?

```
Analogie du firewall :
━━━━━━━━━━━━━━━━━━━━━━

Votre lab = Un château fort
Firewall = Les gardes aux portes

Les gardes vérifient :
• Qui essaie d'entrer (IP source)
• Par quelle porte (port)
• Avec quel laissez-passer (protocole)
• À quelle heure (règles temporelles)

Décision : ✅ Autoriser ou ❌ Bloquer
```

### Types de firewall dans votre lab

```
Architecture firewall multicouche :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Internet
    ↓
[1. Firewall Routeur/Box]
    ↓
[2. Firewall Linux (iptables/ufw)]
    ↓
[3. Network Policies Kubernetes]
    ↓
[4. Security Groups (cloud-like)]
    ↓
Applications

Chaque couche = Protection supplémentaire
```

### Stratégies de firewall

```
Deux approches principales :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Blocklist (Liste noire) :
   • Tout est autorisé par défaut
   • Bloquer spécifiquement le dangereux
   ❌ Moins sécurisé
   ✅ Plus simple à gérer

2. Allowlist (Liste blanche) :
   • Tout est bloqué par défaut
   • Autoriser spécifiquement le nécessaire
   ✅ Plus sécurisé
   ❌ Plus complexe à gérer

📌 Recommandation : Allowlist pour production
```

## UFW (Uncomplicated Firewall)

### Installation et activation

UFW est le firewall le plus simple pour Ubuntu/Debian :

```bash
# Installation (souvent pré-installé)
sudo apt update
sudo apt install ufw

# État actuel
sudo ufw status

# Configuration par défaut sécurisée
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

⚠️ ATTENTION : Si connexion SSH, autoriser SSH AVANT d'activer !
```

### Configuration de base pour MicroK8s

```bash
# Règles essentielles MicroK8s
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# 1. SSH (si accès distant)
sudo ufw allow 22/tcp comment 'SSH'
# Ou port custom
sudo ufw allow 2222/tcp comment 'SSH Custom'

# 2. HTTP/HTTPS pour Ingress
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# 3. Kubernetes API (optionnel, si accès externe)
sudo ufw allow 16443/tcp comment 'K8s API'

# 4. Activer le firewall
sudo ufw enable

# Vérifier les règles
sudo ufw status verbose
```

### Règles pour réseau local

```bash
# Autoriser tout le réseau local
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Méthode 1 : Subnet complet
sudo ufw allow from 192.168.1.0/24 comment 'LAN'

# Méthode 2 : Machine spécifique
sudo ufw allow from 192.168.1.50 comment 'PC Bureau'

# Méthode 3 : Subnet vers port spécifique
sudo ufw allow from 192.168.1.0/24 to any port 6443 comment 'K8s API LAN only'

# Pour MicroK8s inter-pods communication
sudo ufw allow in on cni0
sudo ufw allow in on vxlan.calico
```

### Règles avancées UFW

```bash
# Règles complexes et conditionnelles
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Limiter les tentatives de connexion (anti brute-force)
sudo ufw limit ssh/tcp comment 'SSH rate limit'

# Autoriser range de ports
sudo ufw allow 30000:32767/tcp comment 'NodePort Range'

# Bloquer une IP spécifique
sudo ufw deny from 185.234.218.0/24 comment 'Blocked subnet'

# Autoriser ping (ICMP)
sudo ufw allow proto icmp comment 'Ping'

# Règle temporaire (supprimer après)
sudo ufw allow from 1.2.3.4 to any port 8080
# Plus tard : sudo ufw delete allow from 1.2.3.4 to any port 8080

# Logging
sudo ufw logging on
sudo ufw logging medium  # low, medium, high, full
```

## Iptables (niveau avancé)

### Comprendre iptables

```
Structure iptables :
━━━━━━━━━━━━━━━━━━━

Tables → Chains → Rules

Tables principales :
• filter : Filtrage standard (INPUT, OUTPUT, FORWARD)
• nat : Translation d'adresses
• mangle : Modification de paquets
• raw : Traitement avant tracking

Flux de décision :
Paquet arrive → PREROUTING → INPUT → Application
                     ↓
                  FORWARD → POSTROUTING → Sort
```

### Commandes iptables essentielles

```bash
# Voir les règles actuelles
━━━━━━━━━━━━━━━━━━━━━━━━━

# Toutes les règles
sudo iptables -L -n -v

# Avec numéros de ligne
sudo iptables -L -n -v --line-numbers

# Table NAT
sudo iptables -t nat -L -n -v

# Sauvegarder configuration
sudo iptables-save > /backup/iptables.rules

# Restaurer configuration
sudo iptables-restore < /backup/iptables.rules
```

### Règles iptables pour MicroK8s

```bash
# Configuration iptables manuelle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# 1. Politique par défaut
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# 2. Autoriser loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# 3. Autoriser connexions établies
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 4. SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 5. HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 6. Kubernetes/Calico (ne pas bloquer)
sudo iptables -A INPUT -i cni0 -j ACCEPT
sudo iptables -A INPUT -i vxlan.calico -j ACCEPT

# 7. Bloquer le reste (déjà fait par politique)
```

### Persistance des règles iptables

```bash
# Rendre les règles permanentes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Méthode 1 : iptables-persistent
sudo apt install iptables-persistent
# Sauvegarde automatique à l'installation

# Sauvegarder manuellement
sudo netfilter-persistent save

# Méthode 2 : Script au démarrage
# /etc/network/if-pre-up.d/iptables
#!/bin/bash
/sbin/iptables-restore < /etc/iptables/rules.v4

# Rendre exécutable
chmod +x /etc/network/if-pre-up.d/iptables
```

## Sécurité au niveau Kubernetes

### Network Policies

Les Network Policies sont le firewall natif de Kubernetes :

```yaml
# Politique de base : Bloquer tout
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Autoriser seulement l'Ingress

```yaml
# Autoriser trafic depuis Ingress Controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
    - podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 8080
```

### Isolation par namespace

```yaml
# Isoler complètement un namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
  # DNS toujours autorisé
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

## Fail2ban pour protection automatique

### Installation et configuration

Fail2ban bloque automatiquement les IPs suspectes :

```bash
# Installation
sudo apt install fail2ban

# Configuration de base
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

### Configuration pour services web

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
# Bannir pour 1 heure
bantime = 3600
# Fenêtre de détection : 10 minutes
findtime = 600
# Max 5 essais
maxretry = 5

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log

[nginx-noscript]
enabled = true
port = http,https
filter = nginx-noscript
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-badbots]
enabled = true
port = http,https
filter = nginx-badbots
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-noproxy]
enabled = true
port = http,https
filter = nginx-noproxy
logpath = /var/log/nginx/error.log
maxretry = 2
```

### Monitoring Fail2ban

```bash
# Status général
sudo fail2ban-client status

# Status d'une jail spécifique
sudo fail2ban-client status sshd

# Débannir une IP
sudo fail2ban-client unban 1.2.3.4

# Voir les IPs bannies
sudo iptables -L f2b-sshd -n

# Logs
sudo tail -f /var/log/fail2ban.log
```

## Protection DDoS

### Limiter les connexions

```bash
# Avec iptables - Limiter les nouvelles connexions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Limite 20 connexions/seconde sur port 80
sudo iptables -A INPUT -p tcp --dport 80 \
  -m connlimit --connlimit-above 20 \
  --connlimit-mask 32 -j REJECT

# Limite 100 connexions simultanées par IP
sudo iptables -A INPUT -p tcp --dport 443 \
  -m connlimit --connlimit-above 100 -j REJECT

# Rate limiting général
sudo iptables -A INPUT -p tcp --dport 80 \
  -m limit --limit 25/minute --limit-burst 100 \
  -j ACCEPT
```

### Protection SYN flood

```bash
# Protection contre SYN flood
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Activer SYN cookies
echo 1 > /proc/sys/net/ipv4/tcp_syncookies

# Limiter SYN
sudo iptables -N syn_flood
sudo iptables -A INPUT -p tcp --syn -j syn_flood
sudo iptables -A syn_flood -m limit --limit 1/s --limit-burst 3 -j RETURN
sudo iptables -A syn_flood -j DROP

# Réduire timeout SYN
echo 30 > /proc/sys/net/ipv4/tcp_fin_timeout
echo 2 > /proc/sys/net/ipv4/tcp_synack_retries
```

### Configuration nginx pour anti-DDoS

```nginx
# /etc/nginx/nginx.conf

http {
    # Limite de requêtes
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Limite de connexions
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    server {
        # Appliquer les limites
        location / {
            limit_req zone=one burst=5;
            limit_conn addr 10;
        }

        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_conn addr 20;
        }
    }
}
```

## Monitoring et alertes

### Logs à surveiller

```bash
# Fichiers de logs importants
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Firewall
/var/log/ufw.log
/var/log/syslog | grep -i iptables

# Connexions
/var/log/auth.log     # SSH
/var/log/nginx/*.log  # Web

# Kubernetes
kubectl logs -n ingress <nginx-pod>
journalctl -u snap.microk8s.daemon-kubelite

# Fail2ban
/var/log/fail2ban.log
```

### Script de monitoring simple

```bash
#!/bin/bash
# monitor-security.sh

echo "=== SECURITY MONITOR ==="
echo "Date: $(date)"
echo ""

echo "=== Connexions actives ==="
netstat -an | grep ESTABLISHED | wc -l

echo ""
echo "=== Top 10 IPs connectées ==="
netstat -an | grep ESTABLISHED | \
  awk '{print $5}' | cut -d: -f1 | \
  sort | uniq -c | sort -rn | head -10

echo ""
echo "=== IPs bannies (Fail2ban) ==="
sudo fail2ban-client status | grep "Jail list" | \
  sed 's/.*://g' | tr ',' '\n' | \
  while read jail; do
    echo "Jail: $jail"
    sudo fail2ban-client status $jail | grep "Banned IP"
  done

echo ""
echo "=== Dernières tentatives SSH ==="
grep "Failed password" /var/log/auth.log | tail -5

echo ""
echo "=== Alertes UFW ==="
grep "UFW BLOCK" /var/log/syslog | tail -5
```

### Alertes automatiques

```bash
# Script d'alerte par email
#!/bin/bash
# alert-intrusion.sh

THRESHOLD=100
CURRENT=$(netstat -an | grep :443 | grep ESTABLISHED | wc -l)

if [ $CURRENT -gt $THRESHOLD ]; then
    echo "ALERTE: $CURRENT connexions sur port 443" | \
    mail -s "Alerte Sécurité MicroK8s" admin@example.com
fi

# Ajouter à crontab pour exécution toutes les 5 minutes
# */5 * * * * /path/to/alert-intrusion.sh
```

## Sécurité des ports exposés

### Scan de sécurité

```bash
# Scanner vos propres ports
━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Installation nmap
sudo apt install nmap

# Scan basique
nmap localhost

# Scan complet
nmap -sV -sC -O -p- localhost

# Scan depuis l'extérieur (demander à un ami)
nmap monlab.com

# Interprétation :
# "open" = Port accessible
# "filtered" = Firewall bloque
# "closed" = Rien n'écoute
```

### Fermer les ports inutiles

```bash
# Identifier les services écoutant
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Linux
sudo netstat -tlnp
sudo ss -tlnp
sudo lsof -i -P -n

# Services courants à désactiver si non utilisés :
sudo systemctl disable --now cups      # Impression
sudo systemctl disable --now avahi-daemon  # mDNS
sudo systemctl disable --now bluetooth

# Vérifier après désactivation
sudo netstat -tlnp
```

## GeoIP blocking

### Bloquer par pays

```bash
# Installation des listes GeoIP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Installer ipset
sudo apt install ipset

# Télécharger listes pays (exemple: Chine, Russie)
wget https://www.ipdeny.com/ipblocks/data/countries/cn.zone
wget https://www.ipdeny.com/ipblocks/data/countries/ru.zone

# Créer ipset
sudo ipset create blocklist hash:net

# Ajouter les IPs
for ip in $(cat cn.zone ru.zone); do
    sudo ipset add blocklist $ip
done

# Bloquer avec iptables
sudo iptables -A INPUT -m set --match-set blocklist src -j DROP

# Sauvegarder ipset
sudo ipset save > /etc/ipset.rules
```

### Script de mise à jour GeoIP

```bash
#!/bin/bash
# update-geoip.sh

COUNTRIES="cn ru kp ir"  # Chine, Russie, Corée Nord, Iran
IPSET_NAME="geo_blocklist"

# Créer/vider ipset
ipset create $IPSET_NAME hash:net -exist
ipset flush $IPSET_NAME

# Télécharger et ajouter
for country in $COUNTRIES; do
    wget -q -O - https://www.ipdeny.com/ipblocks/data/countries/${country}.zone | \
    while read ip; do
        ipset add $IPSET_NAME $ip -exist
    done
done

echo "Blocklist mise à jour: $(ipset list $IPSET_NAME | wc -l) réseaux"
```

## Hardening système

### Sécurisation kernel

```bash
# /etc/sysctl.d/99-security.conf
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# Protection SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 5
net.ipv4.tcp_synack_retries = 2

# Ignorer ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Ignorer pings broadcast
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Protection source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Log Martians (paquets impossibles)
net.ipv4.conf.all.log_martians = 1

# Protection MITM
net.ipv4.conf.all.send_redirects = 0

# Appliquer
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

### Audit de sécurité

```bash
# Installation Lynis
sudo apt install lynis

# Audit système complet
sudo lynis audit system

# Rapport
sudo cat /var/log/lynis-report.dat

# Points à vérifier :
# - Suggestions de hardening
# - Packages obsolètes
# - Configurations faibles
# - Permissions fichiers
```

## Backup et récupération

### Sauvegarde configuration firewall

```bash
#!/bin/bash
# backup-firewall.sh

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/backup/firewall"

mkdir -p $BACKUP_DIR

# UFW
sudo ufw status numbered > $BACKUP_DIR/ufw-status-$DATE.txt
cp /etc/ufw/*.rules $BACKUP_DIR/

# Iptables
sudo iptables-save > $BACKUP_DIR/iptables-$DATE.rules
sudo ip6tables-save > $BACKUP_DIR/ip6tables-$DATE.rules

# Fail2ban
cp -r /etc/fail2ban/ $BACKUP_DIR/fail2ban-$DATE/

# Archive
tar -czf $BACKUP_DIR/firewall-backup-$DATE.tar.gz $BACKUP_DIR/*-$DATE*

echo "Backup créé : $BACKUP_DIR/firewall-backup-$DATE.tar.gz"
```

### Plan de récupération

```bash
# Restauration d'urgence
━━━━━━━━━━━━━━━━━━━━━━━━

# 1. Désactiver tout
sudo ufw disable
sudo systemctl stop fail2ban

# 2. Restaurer iptables
sudo iptables-restore < /backup/iptables.rules

# 3. Restaurer UFW
sudo ufw enable

# 4. Restaurer Fail2ban
sudo cp -r /backup/fail2ban/* /etc/fail2ban/
sudo systemctl start fail2ban

# 5. Vérifier
sudo ufw status
sudo fail2ban-client status
```

## Check-list de sécurité

```
Configuration firewall complète :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Firewall de base :
☐ UFW installé et activé
☐ Politique deny par défaut
☐ Ports 80/443 ouverts
☐ SSH protégé (port custom, key-only)
☐ Logs activés

Protection avancée :
☐ Fail2ban configuré
☐ Rate limiting activé
☐ Protection DDoS
☐ GeoIP blocking (si nécessaire)
☐ Network Policies K8s

Monitoring :
☐ Scripts de surveillance
☐ Alertes configurées
☐ Logs centralisés
☐ Audit régulier (Lynis)

Backup :
☐ Configuration sauvegardée
☐ Plan de récupération testé
☐ Documentation à jour
```

## Erreurs courantes à éviter

```
Pièges fréquents :
━━━━━━━━━━━━━━━━━

1. S'enfermer dehors
   → Toujours garder session SSH ouverte
   → Tester dans autre terminal

2. Bloquer Kubernetes
   → Ne pas bloquer trafic inter-pods
   → Autoriser CNI interfaces

3. Oublier la persistence
   → UFW : automatique
   → iptables : nécessite configuration

4. Trop restrictif
   → Commencer permissif
   → Durcir progressivement

5. Pas de monitoring
   → Logs essentiels
   → Savoir qui se connecte

6. Pas de backup
   → Avant changements majeurs
   → Tester restauration
```

## Points clés à retenir

1. **Sécurité en couches** : Routeur + Linux + Kubernetes
2. **Principe du moindre privilège** : N'ouvrir que le nécessaire
3. **Monitoring actif** : Surveiller logs et connexions
4. **Automatisation** : Fail2ban pour bloquer les attaquants
5. **Documentation** : Noter toutes les règles ajoutées
6. **Test externe** : Vérifier depuis l'extérieur régulièrement

## Résumé

Un firewall bien configuré est essentiel pour protéger votre lab MicroK8s accessible depuis Internet. UFW offre une gestion simple pour débuter, tandis qu'iptables permet un contrôle fin. Fail2ban automatise la défense contre les attaques, et les Network Policies Kubernetes ajoutent une couche de sécurité applicative. Avec ces protections en place, votre lab est prêt à affronter Internet en toute sécurité. La prochaine section vous guidera dans les tests finaux de connectivité externe.

⏭️
