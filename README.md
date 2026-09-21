# 🛡️ OWASP-Modsecurity-CRS-DVWA

Déploiement d'un WAF (ModSecurity v3 + OWASP CRS) devant DVWA sur Kali Linux, avec tests SQLi/XSS avant/après protection. Projet pédagogique — M1 OCC, ENI.

**Auteurs :** RAHERINIRINA Justin Elysa Marius, FANOMEZANTSOA Rantoniaina Harlivah
**Cadre :** M1 OCC, ENI

```bash
git clone https://github.com/<votre-compte>/OWASP-Modsecurity-CRS-DVWA.git
```

---

## 🎯 Objectifs du projet

1. **Déployer DVWA** — installer l'application cible délibérément vulnérable, comme environnement de test d'intrusions.
2. **Lancer l'offensive** — simuler des attaques réelles (injection SQL, XSS) pour valider l'exposition initiale.
3. **Activer ModSecurity** — déployer le WAF comme passerelle de sécurité active sur le serveur web.
4. **Bloquer en temps réel** — appliquer les règles OWASP CRS pour intercepter et neutraliser les menaces.

---

## 🏗️ Architecture

```text
Client (navigateur)
        │
        ▼
┌─────────────────────┐
│      Apache2         │
│  ┌─────────────────┐ │
│  │  ModSecurity v3  │ │  ← inspecte chaque requête HTTP
│  │  + OWASP CRS     │ │     avant qu'elle atteigne DVWA
│  └────────┬─────────┘ │
└───────────┼───────────┘
            ▼
      DVWA (PHP + MariaDB)
```

ModSecurity agit comme un filtre placé entre le client et l'application : chaque requête est analysée par les règles CRS avant d'atteindre DVWA. Une requête jugée malveillante est bloquée (HTTP 403) avant même d'être exécutée par le code applicatif.

---

## 🛠️ Environnement technique

| Composant | Outil et rôle |
|---|---|
| Système d'exploitation | Kali Linux (environnement de test d'intrusions) |
| Serveur applicatif | Apache2 (héberge DVWA) |
| Briques logicielles | PHP & MariaDB (base de données du laboratoire) |
| Moteur WAF | ModSecurity v3 (moteur de filtrage et d'inspection) |
| Jeu de règles | OWASP Core Rule Set — CRS (base de signatures de menaces) |
| Cible de test | DVWA (Damn Vulnerable Web Application) |

⚠️ **DVWA doit impérativement rester isolé en local** (machine virtuelle ou réseau isolé) — c'est une application volontairement vulnérable, jamais exposée sur Internet.

---

## 📁 Structure du dépôt

```text
OWASP-Modsecurity-CRS-DVWA/
├── README.md
├── docs/
│   ├── Tutoriel_Installation_ModSecurity_CRS_DVWA.pdf
│   └── Tutoriel_Installation_ModSecurity_CRS_DVWA.docx
├── config/
│   ├── security2.conf.example      # config Apache pour charger ModSecurity + CRS
│   ├── modsecurity.conf.example     # config principale du moteur ModSecurity
│   └── crs-setup.conf.example       # config du OWASP Core Rule Set (niveau de paranoïa, seuils)
└── logs/
    └── modsec_audit.log.sample      # exemple de log d'interception (SQLi + XSS)
```

⚠️ Le fichier `modsec_audit.log.sample` et les valeurs de `crs-setup.conf.example` (niveau de paranoïa `PL2`, seuils) sont des exemples fournis avec ce dépôt — à remplacer par vos propres sorties et réglages une fois la manipulation réalisée sur votre machine.

---

## 🚀 Installation et configuration

### 1. Prérequis

```bash
sudo apt update
sudo apt install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git
```

### 2. Installer DVWA

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git dvwa
cd dvwa
sudo cp config/config.inc.php.dist config/config.inc.php
```

Éditer `config/config.inc.php` pour renseigner les identifiants de la base de données (utilisateur, mot de passe, nom de la base).

Donner les droits nécessaires :

```bash
sudo chown -R www-data:www-data /var/www/html/dvwa
sudo chmod -R 755 /var/www/html/dvwa
```

Démarrer les services et créer la base :

```bash
sudo service apache2 start
sudo service mariadb start
```

Puis ouvrir `http://127.0.0.1/dvwa/setup.php` dans le navigateur et cliquer sur **Create / Reset Database**.

### 3. Installer ModSecurity et le CRS

```bash
sudo apt install libapache2-mod-security2 modsecurity-crs
```

Ce paquet installe ModSecurity, active automatiquement le module Apache `security2`, et fournit les fichiers du CRS.

Activer le moteur ModSecurity en mode blocage (`On`, et non `DetectionOnly`) dans `/etc/modsecurity/modsecurity.conf` :

```apache
SecRuleEngine On
```

Vérifier que le CRS est bien inclus dans la configuration Apache (`/etc/apache2/mods-available/security2.conf` ou `/etc/apache2/conf-available/modsecurity.conf` selon la distribution) :

```apache
IncludeOptional /usr/share/modsecurity-crs/*.conf
IncludeOptional /usr/share/modsecurity-crs/rules/*.conf
```

Redémarrer Apache :

```bash
sudo systemctl restart apache2
```

### 4. Vérifier l'installation

```bash
sudo systemctl status apache2
tail -f /var/log/apache2/modsec_audit.log
```

Si Apache redémarre sans erreur, ModSecurity est actif. Le fichier de log d'audit affichera chaque requête interceptée en temps réel.

---

## 🧪 Tests réalisés

### Test 1 — Injection SQL (tautologie)

Payload injecté : `1' OR '1'='1`

* **Sans CRS** : accès illégitime total aux tables utilisateurs.
* **Avec CRS** : interception instantanée, erreur **403 Access Denied**.

### Test 2 — Cross-Site Scripting (XSS)

Payload injecté : `<script>alert(1)</script>`

* **Sans CRS** : le script s'exécute, risque de vol de cookies de session.
* **Avec CRS** : la balise `<script>` est détectée et bloquée, sessions protégées.

### Synthèse

| Vecteur d'attaque | Sans CRS | Avec CRS |
|---|---|---|
| SQL Injection | ❌ Vulnérable — exploitation possible | ✔ Bloqué — HTTP 403 |
| XSS | ❌ Exécuté — vol de session possible | ✔ Intercepté — payload neutralisé |
| Niveau de sécurité global | ❌ Faible, fortement exposé | ✔ Résilient face au Top 10 OWASP |

### Exemple de log d'interception

```text
[modsecurity] [client 192.168.1.10] Access denied with code 403
Message: SQL Injection Attack Detected (Tautology logic check)
Rule ID: 941100  --  Severity: CRITICAL
Action: Intercepted (Transaction Dropped)
```

---

## ⚠️ Limites et défis opérationnels

* **Faux positifs** : le CRS peut bloquer des requêtes légitimes d'utilisateurs normaux — une configuration fine est indispensable pour l'adapter au contexte métier.
* **Surcharge processeur mineure** induite par l'analyse syntaxique de chaque requête.
* **Un WAF ne remplace pas des pratiques de code sécurisé (SecDevOps)** — c'est un bouclier actif, complément d'une stratégie de défense en profondeur, pas une solution unique.

---

## 📌 Conclusion

Ce projet démontre, sur un cas concret (DVWA), l'efficacité du couple ModSecurity + OWASP CRS pour transformer une application vulnérable en cible résiliente face aux attaques les plus courantes du Top 10 OWASP (SQLi, XSS), sans modifier une seule ligne du code applicatif.

---

## 📄 Documentation complémentaire

Un tutoriel détaillé pas-à-pas est disponible dans [`docs/Tutoriel_Installation_ModSecurity_CRS_DVWA.pdf`](docs/Tutoriel_Installation_ModSecurity_CRS_DVWA.pdf).
