🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.5 Redirection de ports et NAT

## Introduction

La redirection de ports est le pont entre Internet et votre lab MicroK8s. Sans elle, même avec un domaine parfaitement configuré, personne ne pourra accéder à vos services depuis l'extérieur. Cette section démystifie le NAT (Network Address Translation) et vous guide à travers la configuration de la redirection de ports sur votre routeur.

## Comprendre le NAT et la redirection de ports

### Pourquoi le NAT existe-t-il ?

```
Le problème des adresses IP :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Nombre d'adresses IPv4 : 4,3 milliards
Appareils connectés : >20 milliards

Solution : NAT (Network Address Translation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Internet                    Votre réseau local

82.123.45.67 ←──[Routeur]──→ 192.168.1.1 (routeur)
(IP publique)       NAT       192.168.1.100 (PC MicroK8s)
                              192.168.1.101 (laptop)
                              192.168.1.102 (smartphone)

1 IP publique = Des dizaines d'appareils locaux !
```

### Comment fonctionne le NAT

```
Flux sortant (vous → Internet) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. PC (192.168.1.100) → "Je veux google.com"
        ↓
2. Routeur : "Je remplace 192.168.1.100 par 82.123.45.67"
        ↓
3. Google voit : Requête depuis 82.123.45.67
        ↓
4. Google répond à 82.123.45.67
        ↓
5. Routeur : "C'est pour 192.168.1.100"
        ↓
6. PC reçoit la réponse

✅ Fonctionne automatiquement !
```

### Le problème pour votre lab

```
Flux entrant (Internet → vous) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Utilisateur → "Je veux monlab.com (82.123.45.67)"
        ↓
Routeur : "J'ai reçu une requête... mais pour QUI ?"
        ↓
        🤷 192.168.1.100 ?
        🤷 192.168.1.101 ?
        🤷 192.168.1.102 ?
        ↓
❌ Routeur rejette la connexion

Solution : Redirection de ports !
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"Port 80/443 → Toujours vers 192.168.1.100"
```

## Les ports essentiels

### Comprendre les ports

```
Analogie : Immeuble avec appartements
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

IP = Adresse de l'immeuble
Port = Numéro d'appartement

82.123.45.67:80  = Immeuble 82.123.45.67, Appart 80
82.123.45.67:443 = Immeuble 82.123.45.67, Appart 443

Total : 65,535 ports disponibles (1-65535)
```

### Ports standard pour votre lab

```
Ports essentiels à rediriger :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Port  | Service        | Usage
------|----------------|---------------------------
80    | HTTP           | Trafic web non sécurisé
443   | HTTPS          | Trafic web sécurisé (SSL)
22    | SSH            | Accès terminal (optionnel)
6443  | K8s API        | API Kubernetes (optionnel)

Ports additionnels possibles :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
8080  | HTTP alternatif | Applications spécifiques
8443  | HTTPS alternatif| Services secondaires
3000  | Dev apps        | Grafana, apps Node.js
5432  | PostgreSQL      | Si DB exposée
3306  | MySQL           | Si DB exposée
```

### Sécurité des ports

```
Règles de sécurité :
━━━━━━━━━━━━━━━━━━━

✅ À ouvrir :
• 80 et 443 (web standard)

⚠️ À considérer :
• 22 (SSH) - Seulement si nécessaire
• Changer le port SSH (ex: 2222)

❌ À éviter :
• 3306, 5432 (bases de données)
• 6443 (API K8s) sauf si sécurisé
• Ports de services internes
• Ranges complets (ex: 30000-32767)

Principe : Minimum de ports ouverts !
```

## Accéder à l'interface du routeur

### Trouver l'adresse du routeur

```bash
# Sur Linux/Mac
ip route | grep default
# ou
netstat -rn | grep default
# Résultat typique : default via 192.168.1.1

# Sur Windows
ipconfig
# Chercher "Passerelle par défaut"

# Adresses routeur courantes :
192.168.1.1    # Plus courant
192.168.1.254  # Alternative fréquente
192.168.0.1    # Certains modèles
10.0.0.1       # Réseaux en 10.x
```

### Se connecter au routeur

```
Processus de connexion :
━━━━━━━━━━━━━━━━━━━━━━━

1. Ouvrir navigateur web
2. Taper : http://192.168.1.1
3. Page de login apparaît
4. Entrer identifiants

Identifiants par défaut courants :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• admin / admin
• admin / password
• admin / 1234
• [vide] / admin
• Voir étiquette sous le routeur

⚠️ IMPORTANT : Changer le mot de passe !
```

### Navigation dans l'interface

```
Emplacements typiques (varie selon modèle) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Livebox (Orange) :
Configuration avancée → NAT/PAT → Ajouter

Freebox :
Paramètres → Mode avancé → Redirections

Bbox (Bouygues) :
Configuration → NAT → Port Forwarding

SFR Box :
Réseau → NAT → Redirection de ports

Routeur générique :
Advanced → Port Forwarding / Virtual Server
```

## Configuration de la redirection de ports

### Configuration pour HTTP/HTTPS

```
Configuration standard web :
━━━━━━━━━━━━━━━━━━━━━━━━━━━

Règle 1 - HTTP :
• Nom : MicroK8s-HTTP
• Protocole : TCP
• Port externe : 80
• Port interne : 80
• IP destination : 192.168.1.100
• Actif : ✓

Règle 2 - HTTPS :
• Nom : MicroK8s-HTTPS
• Protocole : TCP
• Port externe : 443
• Port interne : 443
• IP destination : 192.168.1.100
• Actif : ✓

Résultat :
Internet:80 → Routeur → 192.168.1.100:80
Internet:443 → Routeur → 192.168.1.100:443
```

### Configuration alternative (ports non standard)

Si votre FAI bloque les ports 80/443 :

```
Contournement ports bloqués :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Règle alternative HTTP :
• Port externe : 8080
• Port interne : 80
• IP destination : 192.168.1.100

Règle alternative HTTPS :
• Port externe : 8443
• Port interne : 443
• IP destination : 192.168.1.100

Accès :
http://monlab.com:8080
https://monlab.com:8443

💡 Moins élégant mais fonctionnel !
```

### Configuration pour SSH (optionnel)

```
Redirection SSH sécurisée :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Règle SSH :
• Nom : SSH-MicroK8s
• Protocole : TCP
• Port externe : 2222 (non standard)
• Port interne : 22
• IP destination : 192.168.1.100

Connexion :
ssh user@monlab.com -p 2222

⚠️ Sécurité SSH :
• Utiliser clés SSH (pas mot de passe)
• Fail2ban pour bloquer attaques
• Port non standard (pas 22)
```

## IP fixe locale (réservation DHCP)

### Pourquoi fixer l'IP locale ?

```
Problème sans IP fixe :
━━━━━━━━━━━━━━━━━━━━━━

Jour 1 : PC MicroK8s = 192.168.1.100
         Redirection → 192.168.1.100 ✓

Jour 2 : PC redémarre
         PC MicroK8s = 192.168.1.105 (nouvelle IP!)
         Redirection → 192.168.1.100 ❌

Solution : Réservation DHCP
━━━━━━━━━━━━━━━━━━━━━━━━━━
Associe MAC address → IP fixe
```

### Trouver l'adresse MAC

```bash
# Linux
ip addr show
# ou
ifconfig
# Chercher : "link/ether xx:xx:xx:xx:xx:xx"

# Windows
ipconfig /all
# Chercher : "Adresse physique"

# Format MAC :
# 6 paires hexadécimales
# Ex: 00:1B:44:11:3A:B7
```

### Configurer la réservation DHCP

```
Configuration réservation :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Dans interface routeur :
DHCP → Réservation → Ajouter

• Nom : MicroK8s-Server
• Adresse MAC : 00:1B:44:11:3A:B7
• IP réservée : 192.168.1.100
• Actif : ✓

Alternative : IP statique sur la machine
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# /etc/netplan/01-netcfg.yaml (Ubuntu)
network:
  ethernets:
    eth0:
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

## DMZ (Zone Démilitarisée)

### Qu'est-ce que la DMZ ?

```
DMZ en bref :
━━━━━━━━━━━━

DMZ = Redirection TOTALE de TOUS les ports
      vers une machine spécifique

Avantages :
✅ Configuration ultra simple
✅ Tous les ports accessibles
✅ Pas de configuration par service

Inconvénients :
❌ Très dangereux (exposition totale)
❌ Pas de filtrage
❌ Un seul appareil possible

⚠️ À éviter sauf tests temporaires !
```

### Alternative : UPnP

```
UPnP (Universal Plug and Play) :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Permet aux applications de :
• Ouvrir des ports automatiquement
• Fermer les ports après usage

Pour MicroK8s :
❌ Généralement déconseillé
❌ Risque de sécurité
❌ Préférer configuration manuelle
```

## Test de la redirection

### Test depuis l'intérieur

```bash
# Test local (peut être trompeur)
curl http://localhost
curl http://192.168.1.100

# Test avec IP publique (NAT loopback)
curl http://82.123.45.67
# ⚠️ Certains routeurs ne supportent pas

# Test avec nom de domaine
curl http://monlab.com
# Si ça fonctionne en local mais pas depuis
# l'extérieur = problème de redirection
```

### Test depuis l'extérieur

```
Méthodes de test externe :
━━━━━━━━━━━━━━━━━━━━━━━━━

1. Smartphone en 4G (WiFi désactivé) :
   • Navigateur → http://monlab.com
   • Doit afficher votre site

2. Service en ligne :
   • downforeveryoneorjustme.com
   • uptimerobot.com (monitoring)
   • gtmetrix.com (test performance)

3. Ami/collègue :
   • Demander test depuis l'extérieur
   • Différent FAI = meilleur test

4. VPN :
   • Se connecter via VPN
   • Tester l'accès
```

### Outils de vérification des ports

```bash
# Test port ouvert (depuis l'extérieur)

# Méthode 1 : Sites web
# canyouseeme.org
# portchecker.co
# yougetsignal.com

# Méthode 2 : nmap (depuis autre machine)
nmap -p 80,443 monlab.com

# Méthode 3 : telnet
telnet monlab.com 80
# Succès = "Connected to..."
# Échec = "Connection refused" ou timeout

# Méthode 4 : nc (netcat)
nc -zv monlab.com 80
# Succès = "succeeded!"
```

## Problèmes courants et solutions

### Port déjà utilisé

```
Diagnostic port utilisé :
━━━━━━━━━━━━━━━━━━━━━━━━

# Linux - Qui utilise le port 80 ?
sudo netstat -tulpn | grep :80
sudo lsof -i :80

# Windows
netstat -ano | findstr :80

Solutions :
━━━━━━━━━━
1. Arrêter le service conflictuel
2. Changer son port
3. Utiliser port alternatif (8080)

Services courants sur port 80 :
• Apache / Nginx local
• Skype (ancien)
• IIS (Windows)
```

### FAI bloque les ports

```
Symptômes blocage FAI :
━━━━━━━━━━━━━━━━━━━━━━
• Ports ouverts localement ✓
• Redirection configurée ✓
• Test externe échoue ❌

Ports souvent bloqués :
• 80 (HTTP)
• 443 (HTTPS)
• 25 (SMTP)
• 445 (SMB)

Solutions :
━━━━━━━━━━
1. Ports alternatifs (8080, 8443)
2. Reverse proxy (Cloudflare Tunnel)
3. VPN avec port forwarding
4. Changer d'offre FAI (pro)
```

### Double NAT (CG-NAT)

```
Problème Double NAT :
━━━━━━━━━━━━━━━━━━━━

Internet → NAT FAI → NAT Routeur → Vous
         ↑
    Pas d'IP publique !

Détection :
• IP routeur commence par 10.x ou 100.x
• traceroute montre 2+ sauts privés

Solutions :
━━━━━━━━━━
1. Demander IP publique au FAI
2. Cloudflare Tunnel (gratuit)
3. VPS + tunnel SSH
4. Service ngrok (payant)
```

### IP dynamique change

```
Problème IP dynamique :
━━━━━━━━━━━━━━━━━━━━━━

Jour 1 : IP = 82.123.45.67 ✓
Jour 2 : IP = 82.123.99.12
         DNS pointe vers ancienne IP ❌

Solutions :
━━━━━━━━━━
1. DNS dynamique (DuckDNS)
2. Script mise à jour DNS
3. IP fixe (option FAI)
4. Cloudflare + script API

Script surveillance IP :
━━━━━━━━━━━━━━━━━━━━━━━
#!/bin/bash
OLD_IP=$(cat /tmp/current_ip.txt 2>/dev/null)
NEW_IP=$(curl -s ifconfig.me)

if [ "$OLD_IP" != "$NEW_IP" ]; then
    echo "IP changée : $OLD_IP → $NEW_IP"
    echo $NEW_IP > /tmp/current_ip.txt
    # Mettre à jour DNS ici
fi
```

## Configuration avancée

### Port triggering

```
Port Triggering :
━━━━━━━━━━━━━━━━

Ouvre des ports dynamiquement
quand un port spécifique est utilisé

Exemple :
• Sortie sur port 80
  → Ouvre ports 8000-8010 entrants

Usage MicroK8s : Rarement utile
```

### Hairpin NAT (NAT Loopback)

```
Problème Hairpin NAT :
━━━━━━━━━━━━━━━━━━━━━

Vous (192.168.1.50) → monlab.com (82.123.45.67)
                      ↓
                  Routeur devrait rediriger
                      ↓
                  Vers 192.168.1.100

Certains routeurs ne supportent pas !

Solutions :
━━━━━━━━━━
1. Activer NAT Loopback (si disponible)
2. Utiliser DNS local (hosts file)
3. Split DNS (DNS interne différent)
```

### QoS (Quality of Service)

```
Optimisation QoS :
━━━━━━━━━━━━━━━━━

Prioriser trafic MicroK8s :
• Priorité haute : 192.168.1.100
• Bande passante garantie : 50%
• Limite upload/download

Bénéfices :
✅ Performance stable
✅ Moins d'impact autres appareils
✅ Meilleure expérience utilisateur
```

## Sécurité de la redirection

### Bonnes pratiques

```
Checklist sécurité :
━━━━━━━━━━━━━━━━━━━

☐ Minimum de ports ouverts
☐ Pas de DMZ permanente
☐ Mots de passe routeur changés
☐ Firmware routeur à jour
☐ UPnP désactivé
☐ WPS désactivé
☐ Logs activés
☐ Firewall MicroK8s configuré
☐ Fail2ban sur services exposés
☐ Monitoring des connexions
```

### Firewall en cascade

```
Architecture sécurisée :
━━━━━━━━━━━━━━━━━━━━━━━

Internet
    ↓
[Firewall Routeur]
    ↓ (ports 80,443 only)
[Firewall Linux (ufw/iptables)]
    ↓
[Network Policies K8s]
    ↓
[Pod Security]

4 couches de protection !
```

### Monitoring des accès

```bash
# Script monitoring connexions

#!/bin/bash
# monitor-connections.sh

# Connexions actives sur port 80/443
netstat -an | grep -E ':80|:443' | grep ESTABLISHED

# IPs uniques connectées
netstat -an | grep -E ':80|:443' | \
  awk '{print $5}' | cut -d: -f1 | \
  sort | uniq -c | sort -rn

# Logs Nginx Ingress
kubectl logs -n ingress nginx-controller -f | \
  grep -v "kube-probe"

# Alertes connexions suspectes
# (>100 connexions même IP = possible attaque)
```

## Alternatives à la redirection de ports

### Cloudflare Tunnel

```
Cloudflare Tunnel (Argo) :
━━━━━━━━━━━━━━━━━━━━━━━━━━

Avantages :
✅ Pas de ports ouverts
✅ Protection DDoS incluse
✅ Fonctionne derrière CG-NAT
✅ SSL automatique

Installation :
━━━━━━━━━━━━━
1. Compte Cloudflare
2. Installer cloudflared
3. Créer tunnel
4. Pointer vers localhost:80

Trafic : Vous → Cloudflare ← Utilisateurs
         (tunnel sortant)
```

### Ngrok

```
Ngrok pour tests :
━━━━━━━━━━━━━━━━━

# Installation
wget https://bin.equinox.io/c/.../ngrok.zip
unzip ngrok.zip

# Exposition port 80
./ngrok http 80

# URL générée :
https://abc123.ngrok.io → localhost:80

⚠️ Gratuit = URL change
   Payant = URL fixe
```

### VPS + Tunnel SSH

```
Architecture VPS :
━━━━━━━━━━━━━━━━━

VPS (IP publique)
    ↓ SSH Tunnel
Machine locale (MicroK8s)

# Depuis machine locale
ssh -R 80:localhost:80 user@vps-ip

Avantages :
✅ Contrôle total
✅ Pas de limites
❌ Coût VPS (5-10€/mois)
```

## Check-list finale

```
Validation redirection ports :
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Configuration :
☐ IP locale fixée (192.168.1.100)
☐ Ports 80/443 redirigés
☐ Configuration sauvegardée
☐ Mot de passe routeur changé

Tests :
☐ Test local : curl localhost
☐ Test LAN : depuis autre machine
☐ Test externe : smartphone 4G
☐ Vérification : canyouseeme.org

Sécurité :
☐ Firewall activé
☐ Logs configurés
☐ UPnP désactivé
☐ Monitoring en place

Documentation :
☐ Ports ouverts documentés
☐ Configuration screenshotée
☐ Accès routeur noté
```

## Points clés à retenir

1. **NAT = Traduction d'adresses** : Permet partage IP publique
2. **Ports 80/443 suffisent** : Pour la majorité des usages web
3. **IP locale fixe obligatoire** : Sinon redirection cassée
4. **Tester depuis l'extérieur** : Seul vrai test valide
5. **Sécurité en couches** : Routeur + Linux + K8s
6. **Alternatives existent** : Si ports bloqués (Cloudflare, ngrok)

## Résumé

La redirection de ports est le pont entre Internet et votre lab MicroK8s. En configurant correctement les règles NAT sur votre routeur, vous permettez au trafic externe d'atteindre vos services. Associée à une IP locale fixe et aux bonnes pratiques de sécurité, cette configuration rend votre lab accessible mondialement. Dans la prochaine section, nous renforcerons cette configuration avec des règles firewall appropriées.

⏭️
