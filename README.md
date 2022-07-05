# Cours MySQL - Sauvegarde et restauration

Références :

* [Chapter 7 - Backup and Recovery](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html) de la doc officielle MySQL
* Chapitre _Sauvegarde et Restauration_ du livre _MySQL 8 - Administration et optimisation_ (**me demander les exports PDF si besoin**).

## Types de sauvegarde

**Logique vs physique**

* Sauvegarde logique : on extrait de la BDD les requêtes SQL permettant de recréer les tables et leur contenu, les procédures stockées, etc. On obtient donc en sortie un ou des fichiers `.sql`.
* Sauvegarde physique : on effectue une copie brute des fichiers du répertoire de donnés (`/var/lib/mysql`). Contrairement à la sauvegarde logique, on obtient des fichiers binaires.

**Base ouverte vs base fermée**

* Base fermée : le serveur de BDD (au sens logiciel) est éteint/inactif quand on lance la sauvegarde.
* Base ouverte : le serveur de BDD est en fonctionnement

**Complète vs incrémentale**

* Sauvegarde complète : on sauvegarde l'ensemble de la BDD quotidiennement. Si celle-ci est très volumineuse, cela peut d'une part prendre du temps, et d'autre part nécessiter un espace de stockage important (on garde souvent des sauvegardes sur un intervalle de temps, par ex. 30 jours, 90 jours ou plus).
* Sauvegarde incrémentale : on sauvegarde la BDD entière chaque semaine. Puis chaque jour, on ne sauvegarde que les modifications faites depuis la dernière sauvegarde complète.

## Outils

**"Standard" (fournis avec MySQL)**

* `mysqldump` pour les sauvegardes logiques
* `cp`, `tar`, etc. pour les sauvegardes physiques (ne fonctionnera qu'en base fermée)
* MySQL Enterprise Backup (éditions Enterprise uniquement) pour les sauvegardes physiques en base ouverte

**Tiers**

Il existe de nombreux outils tiers, payants ou gratuits, offrant différents niveaux de fonctionnalités (sauvegarde sur serveur distant, chiffrement). Quelques pointeurs [ici](https://www.pcwdld.com/best-mysql-backup-tools).

### `mysqldump`

#### Quelques cas d'usage

**Création d'un utilisateur dédié**

Avant tout, nous pouvons commencer par créer un utilisateur dont le rôle va être dédié uniquement à la sauvegarde. [Référence](https://www.serverlab.ca/tutorials/linux/database-servers/create-a-read-only-backup-account-for-mysql/) (qui montre aussi comment planifier des backups, en lançant périodiquement un script avec `cron`).

1. Se connecter à MySQL : `mysql -uroot -p`
2. Créer l'utilisateur en lui assignant un mot de passe (sur un serveur de production, choisir un mdp plus robuste) : `CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'BackupUserPass';`.
3. Octroyer les privilèges nécessaires à cet utilisateur (lecture et verrouillage des tables) : `GRANT PROCESS, LOCK TABLES, SELECT ON *.* TO 'backup_user'@'localhost';`
4. Quitter la console MySQL (`exit` ou Ctrl-D)

**Création d'un dossier pour le stockage**

En tant qu'utilisateur `debian` :

```
mkdir ~/mysql-backups
```

**Sauvegarder toutes les bases de données**

Ceci étant fait, nous pouvons utiliser `mysqldump` avec le paramètre `--all-databases` (notez la redirection de sortie) :

```
mysqldump -ubackup_user -p --all-databases > ~/mysql-backups/all-databases.sql
ls -lh ~/mysql-backups
```

**Ne sauvegarder qu'une base**

On peut tout simplement spécifier le nom d'une base au lieu de `--all-databases`. Si on veut par exemple sauvegarder la base `testdb` :

```
mysqldump -ubackup_user -p testdb > ~/mysql-backups/testdb.sql
```

(si vous ne savez pas quelle base sauvegarder, vous pouvez lister les BDD existantes en une ligne avec `mysql -uroot -p -e 'SHOW DATABASES'`)

**Ne sauvegarder que la structure**

Si on souhaite sauvegarder la structure d'une BDD mais pas son contenu, on peut utiliser le paramètre `--no-data` :

```
mysqldump -ubackup_user -p testdb --no-data > ~/mysql-backups/testdb-schema.sql
```

**Ne sauvegarder que les données**

Si on ne souhaite pas sauvegarder les requêtes de création des tables, on peut à l'inverse utiliser `--no-create-info` :

```
mysqldump -ubackup_user -p testdb --no-create-info > ~/mysql-backups/testdb-data.sql
```

#### Pour aller plus loin

* Page de manuel : `man mysqldump`
* [mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) dans la doc officielle

### Percona Xtra Backup

Nous avions mentionné, lors du cours précédent, l'existence d'implémentations alternatives au MySQL officiel d'Oracle, plus ou moins compatibles avec ce dernier.

Percona, qui commercialise l'une d'entre elles (Percona Server), fournit également un outil de sauvegarde et restauration puissant, gratuit et open source : Percona XtraBackup (abrégé en PXB).

Entre autres, il permet d'effectuer des sauvegardes physiques, base ouverte.

Références :

* [Page produit sur le site de Percona](https://www.percona.com/software/mysql-database/percona-xtrabackup)
* Livre MySQL 8 Administration > Sauvegarde et Restauration > En pratique > 3. Percona XtraBackup

#### Préalable à l'installation

:warning: Percona XtraBackup a toujours un "train de retard" par rapport aux versions de MySQL d'Oracle.

Ainsi, la version disponible au moment de l'écriture de ce support est la 8.0.28, alors que MySQL Server existe en version 8.0.29. PXB devant avoir un numéro de version égal ou supérieur à MySQL, et la version installée lors du dernier cours étant la 8.0.29, il va nous falloir **désinstaller** MySQL.

Normalement, ces deux commandes feront l'affaire. On vous demandera si vous voulez supprimer toutes les données (ce à quoi vous pourrez répondre oui, sachant que nous n'avons pas de données vraiment importantes, et que vous les avez de toute façon sauvegardées précédemment avec `mysqldump`) :

```
sudo apt remove mysql-common mysql-community-client mysql-community-client-core mysql-community-client-plugins mysql-community-server mysql-community-server-core
sudo apt-get remove --purge mysql-community-client mysql-community-server mysql-common 
```

En tant que `debian`, créez d'abord un dossier où vous téléchargerez la version antérieure (8.0.28) d'Oracle MySQL :

```
mkdir mysql-8.0.28 && cd mysql-8.0.28
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-server_8.0.28-1debian10_amd64.deb-bundle.tar
```

Puis désarchivez le `.tar` (qui contient plusieurs packages `.deb`) :

```
tar xvf mysql-server_8.0.28-1debian10_amd64.deb-bundle.tar
```

Puis installez les packages via `dpkg` :

```
sudo dpkg -i mysql-common_8.0.28-1debian10_amd64.deb \
  mysql-community-client-plugins_8.0.28-1debian10_amd64.deb \
  mysql-community-client-core_8.0.28-1debian10_amd64.deb \
  mysql-community-client_8.0.28-1debian10_amd64.deb \
  mysql-client_8.0.28-1debian10_amd64.deb \
  mysql-community-server-core_8.0.28-1debian10_amd64.deb \
  mysql-community-server-core_8.0.28-1debian10_amd64.deb \
  mysql-community-server_8.0.28-1debian10_amd64.deb \
  mysql-server_8.0.28-1debian10_amd64.deb
```

Si en cours de route vous obtenez une erreur liée à des dépendances non satisfaites, vous pouvez tenter :

```
sudo apt --fix-broken install
```