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

**À la fin du processus, on vous demandera de saisir un mot de passe `root` pour MySQL**, puis un mécanisme de chiffrement (garder l'option par défaut).

#### Téléchargement et installation

Commencez par télécharger le [**manuel** de XtraBackup](https://learn.percona.com/hubfs/Manuals/Percona_Xtra_Backup/Percona-XtraBackup-8.0/PerconaXtraBackup-8.0.28-21.pdf). Il offre des informations plus complètes que la section dédiée du livre _MySQL 8_ des éditions ENI.

Puis créez un répertoire pour stocker les archives de XtraBackup : 

```
mkdir pxb-download && cd pxb-download
```

Téléchargez XtraBackup puis décompressez le bundle `.tar` :

```
wget https://downloads.percona.com/downloads/Percona-XtraBackup-LATEST/Percona-XtraBackup-8.0.28-21/binary/debian/bullseye/x86_64/Percona-XtraBackup-8.0.28-21-r78878e9b608-bullseye-x86_64-bundle.tar
tar xvf Percona-XtraBackup-8.0.28-21-r78878e9b608-bullseye-x86_64-bundle.tar 
```

Installez les dépendances puis les paquets `.deb` extraits du `.tar`. Si en cours de route vous rencontrez une erreur, exécutez `sudo apt --fix-broken install`.

```
sudo apt install libdbd-mysql-perl libcurl4-openssl-dev libcurl4 libev4
sudo dpkg -i percona-xtrabackup-80_8.0.28-21-1.bullseye_amd64.deb percona-xtrabackup-dbg-80_8.0.28-21-1.bullseye_amd64.deb percona-xtrabackup-test-80_8.0.28-21-1.bullseye_amd64.deb 
```

#### Téléchargement d'un programme d'exemple

Pour démontrer les capacités de _hot backup_ (_online backup_ ou sauvegarde base ouverte) de PXB, nous allons utiliser un petit programme très basique, qui insère des lignes en continu dans une table.

Retournez dans votre home directory : `cd`.

Il est écrit en langage Go. Ces commandes vous permettront d'installer Go, Git, de télécharger le code source du programme, de le compiler :

```
sudo apt install -y golang git
git clone https://github.com/bhubr/golang-mysql-example.git
cd golang-mysql-example
go build
```

Puis vous pourrez initialiser la BDD `testdb` utilisée par ce programme, et créer un utilisateur MySQL dédié, en exécutant un script d'initialisation :

```
mysql -uroot -p < init.sql
```

Enfin, vous pourrez exécuter le programme :

```
./mysql
```

Vous verrez des lignes semblables à celles-ci s'afficher :

```
inserted 00000000, affected: 1
inserted 00000001, affected: 1
inserted 00000002, affected: 1
inserted 00000003, affected: 1
inserted 00000004, affected: 1
inserted 00000005, affected: 1
...
```

**Laissez le programme fonctionner** et ouvrez une deuxième session SSH (une alternative serait d'utiliser un multiplexeur de terminaux comme `screen` ou `tmux`).

#### Sauvegarde

> Voir le chapitre 13 du [manuel de PXB](https://learn.percona.com/hubfs/Manuals/Percona_Xtra_Backup/Percona-XtraBackup-8.0/PerconaXtraBackup-8.0.28-21.pdf) pour un exemple complet de cycle de sauvegarde/restauration.

Comme nous l'avions fait avant d'utiliser `mysqlbackup`, nous allons créer un utilisateur dédié à la sauvegarde via XtraBackup. Dans la console MySQL :

```
mysql> CREATE USER 'xbackup'@'localhost' IDENTIFIED BY 'MyPassword'; 
mysql> GRANT BACKUP_ADMIN, RELOAD, LOCK TABLES, REPLICATION CLIENT, PROCESS ON *.* TO 'xbackup'@'localhost'; 
mysql> GRANT SELECT ON performance_schema.log_status TO 'xbackup'@'localhost';
mysql> GRANT SELECT ON performance_schema.keyring_component_status TO xbackup@'localhost';
mysql> FLUSH PRIVILEGES;
```

Puis, une fois ressorti de la console MySQL : `sudo mkdir /opt/pxb-backups && sudo chown mysql:mysql /opt/pxb-backups`.

Puis lancez la commande de backup (on vous demandera le mot de passe de l'utilisateur `xbackup`) :

```
sudo mkdir /opt/pxb-backups/full && sudo chown mysql:mysql /opt/pxb-backups/full
sudo -u mysql xtrabackup --user=xbackup --password --backup --target-dir=/opt/pxb-backups/full/
```

**Cette commande sauvegardera toutes les bases**. Alternativement, on peut spécifier une liste de BDD, séparées par des espaces, entre quotes :

```
sudo mkdir /opt/pxb-backups/partial && sudo chown mysql:mysql /opt/pxb-backups/partial
sudo -u mysql xtrabackup --user=xbackup --password --backup --databases='testdb' --target-dir=/opt/pxb-backups/partial/
```

> **Retournez dans la session où tourne le programme en Go** pour l'**interrompre** avec Ctrl-C.

#### Restauration

La restauration d'un backup se fait en deux étapes : une étape de "préparation", puis la restauration proprement dite.

Si on devait restaurer toutes les bases sauvegardées dans `/opt/pxb-backups/full` vers une **nouvelle** installation de MySQL (ou en ayant supprimé le datadir de l'actuelle), on procéderait ainsi (serveur MySQL éteint) :

```
xtrabackup --prepare --target-dir=/opt/pxb-backups/full/
sudo -u mysql xtrabackup --copy-back --target-dir=/opt/pxb-backups/full/ --datadir=/var/lib/mysql
```

#### Exercices

Suivant le point où le groupe est rendu.

* Suivez les instructions du chapitre 14 du manuel de PXB pour créer et restaurer des **backups incrémentaux** (vous aurez probablement besoin d'effacer la BDD sauvegardée si vous souhaitez la restaurer ensuite).
* Suivez les instructions du chapitre 15 pour effectuer un backup compressé. Vous aurez besoin de l'outil `percona-release` dont les instructions de téléchargement/installation se trouvent ici : <https://docs.percona.com/percona-software-repositories/installing.html>


