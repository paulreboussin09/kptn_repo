# InfraWeb — Apprendre Linux en ligne de commandes avec Docker

Ce dépôt sert de support pédagogique pour apprendre à administrer un serveur Linux via la ligne de commandes, en utilisant Docker comme environnement isolé et reproductible.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Machine hôte                       │
│                                                     │
│  localhost:8080 ──► [  web  ] Ubuntu + Apache + PHP │
│  localhost:8081 ──► [ phpmyadmin ]                  │
│                          │                          │
│                     réseau interne                  │
│                          │                          │
│                      [  db  ] MySQL 8               │
│                                                     │
│  ./www/ ──────────► /var/www/html  (volume partagé) │
└─────────────────────────────────────────────────────┘
```

---

## Prérequis

- [Docker Desktop](https://docs.docker.com/get-docker/) installé sur votre machine (inclut Docker Compose)

---

## Prérequis spécifiques Windows

Docker Desktop utilise **WSL2** (Windows Subsystem for Linux) comme moteur sur Windows. Il le configure automatiquement, mais deux étapes préalables sont nécessaires.

### Étape 1 — Activer WSL2

Ouvrir **PowerShell en administrateur** et exécuter :

```powershell
wsl --install
```

Redémarrer la machine si demandé.

> Sur Windows 10/11 Home, WSL2 est obligatoire car Hyper-V n'est pas disponible. Sur Pro/Enterprise, WSL2 reste la solution recommandée.

### Étape 2 — Installer Docker Desktop

Télécharger et installer [Docker Desktop pour Windows](https://www.docker.com/products/docker-desktop/).

Lors de l'installation, laisser cochée l'option **"Use WSL 2 instead of Hyper-V"**.

### Vérifier que tout fonctionne

Ouvrir **PowerShell** (sans administrateur) et taper :

```powershell
docker --version
```

Si une version s'affiche, Docker est prêt. Vous pouvez continuer avec la suite du TP.

---

## 1. Lancer les conteneurs

```bash
docker compose up -d
```

Cette commande démarre les trois conteneurs en arrière-plan :

| Conteneur | Rôle | Accès |
|---|---|---|
| `infraweb` | Serveur Ubuntu (Apache + PHP) | http://localhost:8080 |
| `infraweb-db` | Base de données MySQL | interne uniquement |
| `infraweb-phpmyadmin` | Interface graphique MySQL | http://localhost:8081 |

Pour entrer dans le serveur web et obtenir un shell interactif :

```bash
docker exec -it infraweb bash
```

Vous êtes maintenant dans un terminal Linux, à l'intérieur du conteneur. Les commandes suivantes sont à exécuter **dans ce terminal**.

---

## 2. Commandes utiles
### 2.1 Mettre à jour les paquets
Avant d'installer quoi que ce soit, il est bon de mettre à jour la liste des paquets disponibles :

```bash
apt update
```

> `apt` est le gestionnaire de paquets d'Ubuntu. `update` rafraîchit la liste des paquets depuis les dépôts officiels.

### 2.2 manipulations fréquentes

```bash
uname -a         # lister informations systèmes
whoami           # afficher l'identifiant de l'utilisateur courant
ls -al           # lister les fichiers du dossier
cd /var/www      # se déplacer dans un dossier
pwd              # afficher le chemin absolu du dossier courant                 
mkdir cours      # créer un dossier appelé "cours"
touch index.html # créer un fichier appelé "index.html"
rm fichier.txt   # supprimer un fichier appelé "fichier.txt"
rm -d cours      # supprimer un dossier appelé "cours"
```

## 3. Installer Apache2

```bash
apt install -y apache2
```

> Le flag `-y` répond automatiquement "oui" aux confirmations d'installation.

Vérifier qu'Apache est bien installé :

```bash
apache2 -v
```

---

## 4. Installer PHP

PHP est le langage de script côté serveur le plus utilisé pour le web. On installe PHP avec le module Apache et quelques extensions courantes :

```bash
apt install -y php libapache2-mod-php php-mysql
```

- `php` : l'interpréteur PHP
- `libapache2-mod-php` : le module qui permet à Apache d'exécuter du PHP
- `php-mysql` : l'extension PHP pour communiquer avec MySQL

Vérifier l'installation :

```bash
php -v
```

---

## 5. Démarrer le service Apache

Dans un conteneur Docker, les services ne démarrent pas automatiquement. Il faut les lancer manuellement :

```bash
service apache2 start
```

Vérifier que le service tourne :

```bash
service apache2 status
```

---

## 6. Tester la page par défaut

Depuis votre navigateur sur votre machine hôte, ouvrez :

```
http://localhost:8080
```

Vous devriez voir la page `index.html` qui se trouve dans le dossier `./www/` sur votre machine.

---

## 7. La persistance des fichiers

Le dossier `./www/` sur votre machine est **directement synchronisé** avec `/var/www/html/` dans le conteneur. C'est ce qu'on appelle un **volume bind mount**.

```
[votre machine]  ./www/index.html
        ↕  (synchronisé en temps réel)
[conteneur]      /var/www/html/index.html
```

Cela signifie que :
- Vous pouvez éditer les fichiers avec votre éditeur préféré sur votre machine
- Les modifications sont immédiatement visibles dans le conteneur
- Les fichiers **ne sont pas perdus** quand le conteneur redémarre

Afficher le contenu du dossier web depuis le conteneur :

```bash
ls -la /var/www/html/
```

---

## 8. Créer une page PHP

Créer un fichier PHP pour tester que PHP fonctionne bien :

```bash
cat > /var/www/html/info.php << 'EOF'
<?php
phpinfo();
EOF
```

Ouvrir dans le navigateur :

```
http://localhost:8080/info.php
```

Vous devriez voir la page d'informations PHP avec toutes les extensions installées.

> Pensez à supprimer ce fichier en production — il expose des informations sensibles sur le serveur.

```bash
rm /var/www/html/info.php
```

---

## 9. Se connecter à la base de données MySQL

### Depuis phpMyAdmin (interface graphique)

Ouvrir dans le navigateur :

```
http://localhost:8081
```

Se connecter avec :
- **Utilisateur** : `etudiant`
- **Mot de passe** : `etudiant`

### Depuis le conteneur web (ligne de commandes)

Installer le client MySQL :

```bash
apt install -y mysql-client
```

Se connecter à la base de données (le nom `db` est le nom du service Docker) :

```bash
mysql -h db -u etudiant -p etudiant mabase
```

Quelques commandes SQL de base :

```sql
-- Lister les bases de données
SHOW DATABASES;

-- Créer une table
CREATE TABLE utilisateurs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100)
);

-- Insérer une ligne
INSERT INTO utilisateurs (nom, email) VALUES ('Alice', 'alice@example.com');

-- Lire les données
SELECT * FROM utilisateurs;

-- Quitter
EXIT;
```

---

## 10. Créer une page PHP connectée à MySQL

```bash
cat > /var/www/html/db-test.php << 'EOF'
<?php
$connexion = new PDO('mysql:host=db;dbname=mabase', 'etudiant', 'etudiant');
$resultat = $connexion->query('SELECT * FROM utilisateurs');

foreach ($resultat as $ligne) {
    echo $ligne['nom'] . ' — ' . $ligne['email'] . '<br>';
}
EOF
```

Ouvrir dans le navigateur :

```
http://localhost:8080/db-test.php
```

---

## 11. Les permissions fichiers

Sur Linux, chaque fichier appartient à un utilisateur et un groupe. Apache tourne sous l'utilisateur `www-data`.

Voir les permissions du dossier web :

```bash
ls -la /var/www/html/
```

Changer le propriétaire d'un fichier :

```bash
chown www-data:www-data /var/www/html/index.html
```

Changer les permissions d'un fichier :

```bash
# Donne les droits lecture/écriture au propriétaire, lecture seule aux autres
chmod 644 /var/www/html/index.html
```

> En production, les fichiers web ne doivent **jamais** être en 777 (tout le monde peut écrire).

---

## 12. Lire les logs Apache

Les logs sont essentiels pour diagnostiquer des problèmes. Apache écrit deux fichiers de log :

```bash
# Log des accès (toutes les requêtes HTTP reçues)
tail -f /var/log/apache2/access.log

# Log des erreurs (erreurs PHP, fichiers manquants, etc.)
tail -f /var/log/apache2/error.log
```

> `tail -f` affiche les dernières lignes en temps réel. Appuyez sur `Ctrl+C` pour quitter.

---

## 13. Arrêter et supprimer les conteneurs

Quitter le shell du conteneur :

```bash
exit
```

Arrêter les conteneurs :

```bash
docker compose down
```

> Les fichiers dans `./www/` et les données MySQL sont conservés grâce aux volumes.

Pour tout supprimer (conteneurs **et** données MySQL) :

```bash
docker compose down -v
```

---

## Récapitulatif des commandes

| Action | Commande |
|---|---|
| Démarrer tous les conteneurs | `docker compose up -d` |
| Entrer dans le serveur web | `docker exec -it infraweb bash` |
| Mettre à jour les paquets | `apt update` |
| Installer Apache | `apt install -y apache2` |
| Installer PHP | `apt install -y php libapache2-mod-php php-mysql` |
| Installer le client MySQL | `apt install -y mysql-client` |
| Démarrer Apache | `service apache2 start` |
| Voir le statut d'Apache | `service apache2 status` |
| Lire les logs en temps réel | `tail -f /var/log/apache2/access.log` |
| Voir les permissions | `ls -la /var/www/html/` |
| Changer les permissions | `chmod 644 fichier` |
| Changer le propriétaire | `chown www-data:www-data fichier` |
| Se connecter à MySQL | `mysql -h db -u etudiant -petudiant mabase` |
| Arrêter les conteneurs | `docker compose down` |
| Arrêter et tout supprimer | `docker compose down -v` |
