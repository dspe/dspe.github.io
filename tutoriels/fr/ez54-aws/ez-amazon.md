eZ 5.4 cluster sur Amazon AWS
=============================

Ce tutoriel va se consacrer à la mise en place d'un cluster eZ Publish 5.4 sur l'architecture Cloud d'Amazon AWS.

Petit rappel sur le cluster d'eZ; depuis les versions 5.x, seul le **cluster eZDFS** est supporté par les deux noyaux. Afin de résumer au mieux, ce cluster s'appuie sur un **lien NFS** depuis les serveurs frontaux vers un **NAS/SAN** mais également deux tables de données contenant des informations sur les caches et images clusterisés.

Pour plus d'informations consultez le site de la [documentation d'eZ](https://doc.ez.no/eZ-Publish/Technical-manual/5.x/Features/Clustering/Cluster-File-Handlers#eztoc132713_4).

Architecture Amazon
-------------------

Nous allons mettre en place une architecture de hautes disponibilités à savoir:
- un loadbalancer
- deux serveurs [Varnish Cache](https://www.varnish-cache.org/)
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

L'arrivée de la version 5.4 d'eZ Publish apporte le support natif du cluster via la bundle ```oneupFlysystemBundle```. Ce bundle permet de pousser vos fichiers vers du Dropbox, Amazon S3, Google Drive, etc.

> Attention: le noyau d'eZ Publish impose l'utilisation de la version 0.4 contenant tous les adapters de la documentation contrairement à la version 1.x

Afin de configurer le bundle pour utiliser l'adapter AwsS3, nous allons avoir besoin du SDK PHP que nous fournis Amazon. Vous allez voir que les problèmes commencent ;) Installons le SDK:

```
composer require "aws/aws-sdk-php:2.*"
```

Nous allons placer des variables d'environnement dans le virtual host. Vous allez voir que le SDK est capricieux avec nous si nous n'agissons pas de cette manière.

Première action mettre à jour le VirtualHost (ici nous utilisons Apache. A adapter en fonction de votre configuration).

```
SetEnv AWS_ACCESS_KEY_ID TheAccessKeyIdValue
SetEnv AWS_SECRET_ACCESS_KEY TheSecretAccessKey
SetEnv AWS_S3_REGION eu-west-1
SetEnv AWS_S3_BUCKET my-app-bucket
SetEnv AWS_S3_PREFIX var/ezdemo/storage/images
SetEnv AWS_S3_PUBLIC_HOST s3-eu-west-1.amazonaws.com
```

Les noms ```AWS_ACCESS_KEY_ID``` et ```AWS_SECRET_ACCESS_KEY``` ne peuvent être changés. Vous pouvez trouver plus d'informations [ici](http://docs.aws.amazon.com/aws-sdk-php/v2/guide/credentials.html#using-credentials-from-environment-variables).

Créons le fichier ```ezpublish/config/parameters_prod.php``` avec le contenu suivant:

```php
<?php

// AWS S3 storage credentials
$container->setParameter('aws_s3_key',    getenv('AWS_ACCESS_KEY_ID'));
$container->setParameter('aws_s3_secret', getenv('AWS_SECRET_ACCESS_KEY'));
$container->setParameter('aws_s3_region', getenv('AWS_S3_REGION'));
$container->setParameter('aws_s3_bucket', getenv('AWS_S3_BUCKET'));
$container->setParameter('aws_s3_prefix', getenv('AWS_S3_PREFIX'));
$container->setParameter('aws_s3_host',   getenv('AWS_S3_PUBLIC_HOST'));
```

N'oubliez pas de l'importer depuis le fichier ```config_prod.yml```.

Il ne reste plus qu'à configurer ezpublish et le composant IO pour terminer notre voyage ;) Dans le fichier ```ezpublish/config/ezpublish_prod.yml```. Pour des informations complémentaires sur la configuration complète se trouve sur [doc.ez.no](https://doc.ez.no/display/EZP/Persistence+cache+configuration).

Dans le fichier placer la configuration suivante:

```
ez_io:
    metadata_handlers:
        default:
            legacy_dfs_cluster:
                connection: doctrine.dbal.dfs_connection
    binarydata_handlers:
        default:
            flysystem:
                adapter: default
ezpublish:
    system:
        default:
            io:
                metadata_handler: default
                binarydata_handler: default
                url_prefix: "//%aws_s3_host%/%aws_s3_bucket%/%aws_s3_prefix%"

oneup_flysystem:
    adapters:
        default:
            awss3:
                client: s3_client
                bucket: %aws_s3_bucket%
                prefix: %aws_s3_prefix%

services:
    s3_client:
        class: Aws\S3\S3Client
        factory_class: Aws\S3\S3Client
        factory_method: factory
        arguments:
            -
                region: "%aws_s3_region%"
                credentials:
                    key: "%aws_s3_key%"
                    secret: "%aws_s3_secret%"
```

Vous trouverez la documentation pour l'[adapter AWS S3](https://github.com/1up-lab/OneupFlysystemBundle/blob/0.x-dev/Resources/doc/adapter_awss3.md) sur github.

Il ne reste plus qu'à tester le bon fonctionnement.
