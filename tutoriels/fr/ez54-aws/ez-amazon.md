eZ 5.4 cluster sur Amazon AWS
=============================

Ce tutoriel va se consacrer à la mise en place d'un cluster eZ Publish 5.4 sur l'architecture Cloud d'Amazon AWS.

Petit rappel sur le cluster d'eZ. Depuis les versions 5.x, seul le **cluster eZDFS** est supporté par les deux noyaux. Afin de résumer au mieux, ce cluster s'appuie sur un lien NFS depuis les serveurs frontaux vers un NAS/SAN mais également deux tables de données contenant des informations sur les caches et images clusterisés. Pour plus d'informations consultés le site de la [documentation d'eZ](https://doc.ez.no/eZ-Publish/Technical-manual/5.x/Features/Clustering/Cluster-File-Handlers#eztoc132713_4).

Architecture Amazon
-------------------

Nous allons mettre en place une architecture de hautes disponibilités à savoir:
- un loadbalancer
- deux serveurs Varnish Cache
- deux serveurs web eZ
- un [RDS](https://aws.amazon.com/fr/rds/) (Relational Database Service)
- un [ElastiCache](https://aws.amazon.com/fr/elasticache/) pour la mise en place des sessions et cache provenant de stash.
- un [bucket S3](https://aws.amazon.com/s3/) qui contiendra nos fichiers clusterisés

Je ne détaillerais pas l'installation des serveurs web et d'eZ; je considère que vous savez déjà faire toute cette partie.

A noter: pour la configuration des droits sur le bucket, vous pouvez vous inspirer de la configuration suivante:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "StatementS3Objects",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::mybucket/*"
            ]
        },
        {
            "Sid": "StatementS3Bucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::mybucket"
            ]
        },
        {
            "Sid": "StatementS3",
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListAllMyBuckets"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ]
        }
    ]
}
```

L'utilisateur peut donc lister tous ces buckets, lister le contenu du bucket "mybucket", et avoir le contrôle complet dessus.

Configuration pour ElastiCache
------------------------------

Cette première partie va vous permettre de mettre en cache les informations du [Service Provider Interface](https://doc.ez.no/display/EZP/Persistence+cache) (SPI) d'eZ.
Il faut alors dans un premier temps installer le plugin **memcached** sur votre environnement.

> Attention: Amazon fourni un package amélioré permettant l'auto discovery du cluster.

Pour la documentation complète du plugin à installer sur php 5.x ou php 7.x, veuillez vous référer à la [documentation](http://docs.aws.amazon.com/fr_fr/AmazonElastiCache/latest/UserGuide/Appendix.PHPAutoDiscoverySetup.html)

Pour profiter pleinement d'eZ et d'ElastiCache, il faut à la fois configurer Stash et les sessions PHP.

> Il est possible de configurer les sessions PHP via Symfony ou en modifiant le fichier php.ini

Pour des raisons de simpliciter la configuration via le fichier *php.ini* sera suffisante. Nous allons donc appliquer les paramètres suivants dans ```/etc/php5/fpm/php.ini``` (dans le cas ou vous utilisez php-fpm bien entendu).

```
session.save_handler = "memcached"
session.save_path = "php-session.xxxxxxx.0001.apne1.cache.amazonaws.com:11211"
```

et pour la configuration de stash:

```yaml
stash:
    caches:
        default:
            drivers: [ Memcache ]
            inMemory: true
            registerDoctrineAdapter: false
            Memcache:
                prefix_key: ezdemo_
                retry_timeout: 1
                servers:
                    -
                        server: php-session.xxxxxxx.0001.apne1.cache.amazonaws.com
                        port: 11211
```

Vous pouvez tester à ce niveau là, afin de vérifier le bon fonctionnement du site.

Configuration du cluster Legacy
-------------------------------

Il ne faut pas oublier que vous avez besoin de configurer les deux noyaux afin de s'assurer du fonctionnement aux petits oignons de toute l'application: back-office et front-office.
Le support du cluster AWS-S3 n'est par défaut pas inclus dans le noyau Legacy, mais grâce à [Bertrand Dunogier](http://share.ez.no/community/profile/10106) ceci est devenu réalité. Passons à l'installation:

```
composer require "ezsystems/ezdfs-fsbackend-dispatcher:~1.0@beta"
composer require ezsystems/ezdfs-fsbackend-aws-s3:~1.0@beta
```

On en a fini avec l'installation dans le Legacy. Place à la configuration.

> A noter: Il est nécessaire de créer un dossier 'NFS' mais un simple dossier est suffisant; en d'autres termes, seul la création d'un dossier est nécessaire.

Commençons par configurer le fichier ```settings/override/file.ini.append.php``` comme suit:

```
[ClusteringSettings]
FileHandler=eZDFSFileHandler

[eZDFSClusteringSettings]
MountPointPath=/media/nfs
DFSBackend=eZDFSFileHandlerDFSDispatcher
DBHost=cluster_server
DBName=db_cluster
DBUser=root
DBPassword=root
MetaDataTableNameCache=ezdfsfile_cache

[DispatchableDFS]
DefaultBackend=eZDFSFileHandlerDFSBackend

PathBackends[var/ezdem_site/storage/images]=eZDFSFileHandlerDFSAmazon
```

> N'oubliez pas de créer le dossier /media/nfs

Seconde étape, la configuration du handler amazon: pour celà nous allons créer le fichier ```settings/override/dfsamazons3.ini.append.php```:

```php
<?php /*

[BackendSettings]
AccessKeyID=
SecretAccessKey=
Bucket=
Region=
```

A partir de ce point, il est alors possible de suivre la documentation sur la mise en place du [cluster eZDFS](https://doc.ez.no/eZ-Publish/Technical-manual/5.x/Features/Clustering/Setting-it-up-for-an-eZDFSFileHandler), à savoir:

### Création du script pour desservir les images

Dans le dossier ```ezpublish_legacy``` copier ou renommer le fichier ```config.php-RECOMMENDED``` en ```config.php```. Il suffit simplement d'activer les lignes définissant les paramètres de connection:

```php
define( 'CLUSTER_STORAGE_BACKEND', 'dfsmysqli' );
define( 'CLUSTER_STORAGE_HOST', 'mysql_rds_domainname' );
define( 'CLUSTER_STORAGE_PORT', 3306 );
define( 'CLUSTER_STORAGE_USER', 'dbuser' );
define( 'CLUSTER_STORAGE_PASS', 'dbpassword' );
define( 'CLUSTER_STORAGE_DB', 'ezpcluster' );
define( 'CLUSTER_STORAGE_CHARSET', 'utf8' );
define( 'CLUSTER_MOUNT_POINT_PATH', '/media/nfs' );

// New metadata configurations introduced in eZ Publish 5.2
define( 'CLUSTER_METADATA_TABLE_CACHE', 'ezdfsfile_cache' );
define( 'CLUSTER_METADATA_CACHE_PATH', '/cache/' );
define( 'CLUSTER_METADATA_STORAGE_PATH', '/storage/' );
```

### Importer les tables SQL dans la base de donnée 'cluster'

Le script SQL se trouve dans le dossier ```ezpublish_legacy/kernel/sql/mysql/cluster_dfs_schema.sql```.

### Importer les fichiers dans le cluster

Placez vous dans le dossier ```ezpublish_legacy```, vérifier bien vos paramètres de connection de base de données pour le cluster mais également d'eZ Publish, vous êtes alors prêt pour clusteriser:

```
php bin/php/clusterize.php --siteaccess <admin_siteaccess>
```

Vous pouvez vérifier au même moment si des informations s'enregistrent:
* dans les tables de cluster
* sur le bucket

Si le bucket est vide, jettez un oeil au fichier d'erreur du legacy ```ezpublish_legacy/var/logs/error.log```.

Tout fonctionne? Bonne nouvelle, nous pouvons passer à l'étape suivante.

Configuration du cluster eZ 5
-----------------------------
