Publiez vos articles eZ sur Twitter
===================================

A travers ce tutoriel, je vais vous montrer comment publier automatiquement vos articles vers un compte twitter associé. Rien de très compliqué mais cela permettra de vous faire découvrir les **Signal Slots**.
Vous trouverez sur github une version alpha de ce bundle. N'hésitez pas à contribuer si le coeur vous en dit :)

Mise en place
-------------

Twitter vous propose une API richement fournie en fonctionnalités. Certains peuvent être lancer sans avoir besoin de se connecter mais pour d'autres, dont la publication, il vous faudra être connecté à cet API. La documentation complète de l'API se trouve à [cette adresse](https://dev.twitter.com/overview/documentation).

Avant toutes choses, il vous faudra créer une application twitter via https://apps.twitter.com/. Le gros avantage, et ce qui va nous faire gagner du temps, est que le dashboard va nous permettre de générer un **access token** et un **access token secret** pour se connecter.
Il vous faudra mettre dans un coin les informations suivantes:
- Consummer API
- Consumer Secret
- Access token
- Access token secret

Nous allons rajouter la configuration suivante dans le fichier ```app/config/config.yml```:
```yaml
pvr_ezsocial:
    networks:
        twitter:
            consumer_key: "abcdefghijklmnopqrstuvwxyz"
            consumer_secret: "abcdefghijklmnopqrstuvwxyz1234567890"
            access_token: "abcdefghijklmnopqrstuvwxyz-123456"
            access_secret: "01234567890"
```

Le bundle pour gérer plusieurs types de réseaux sociaux mais également de publier en fonction des types de contenus etc.

Nous allons créer un service comprenant la gestion de notre réseau social afin de l'appeler depuis notre ```SignalSlot```.

```php
<?php

namespace Pvr\EzSocialBundle\Networks\Handler;

use Abraham\TwitterOAuth\TwitterOAuth;
use Psr\Log\LoggerInterface;
use Pvr\EzSocialBundle\Networks\NetworkInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class TwitterHandler implements NetworkInterface
{
    /**
     * @var TwitterOAuth
     */
    private $connection;
    private $container;
    protected $consumer_key;
    protected $consumer_secret;
    protected $access_token;
    protected $access_secret;
    private $logger;

    /**
     * TwitterHandler constructor.
     *
     * @param ContainerInterface $container
     * @param LoggerInterface $logger
     */
    public function __construct(ContainerInterface $container, LoggerInterface $logger)
    {
        $networks = $container->getParameter('pvr_ezsocial.networks');
        $this->consumer_key     = $networks['twitter']['consumer_key'];
        $this->consumer_secret  = $networks['twitter']['consumer_secret'];
        $this->access_token     = $networks['twitter']['access_token'];
        $this->access_secret    = $networks['twitter']['access_secret'];
        $this->logger = $logger;
        $this->container = $container;
        $this->connect();
    }

    /**
     * Connect to Twitter Oauth API
     */
    public function connect()
    {
        if (empty($this->access_token) || empty($this->access_secret)) {
            // @TODO
        }
        // Create the connection
        $this->connection = new TwitterOAuth(
            $this->consumer_key,
            $this->consumer_secret,
            $this->access_token,
            $this->access_secret
        );
    }

    /**
     * @param $parameters
     */
    public function publish($parameters)
    {
        if (!isset($parameters['status'])) {
            $this->logger->debug('Twitter: No status parameters');
            exit;
        }

        if (!isset($parameters['locationId'])) {
            $this->logger->debug('Twitter: No Location ID found');
            exit;
        }

        $location = $this->container->get('ezpublish.api.repository')
            ->getLocationService()->loadLocation($parameters['locationId']);

        $locationUrl = $this->container->get('router')->generate(
            $location,
            ['siteaccess' => $parameters['siteaccess']],
            UrlGeneratorInterface::ABSOLUTE_URL
        );

        $this->connection->post("statuses/update", [
            "status" => $parameters['status'] . " " . $locationUrl
        ]);

        if ($this->connection->getLastHttpCode() != 200) {
            // Handle error case
            $this->logger->debug('Twitter: error connection: ' . $this->connection->getLastHttpCode() . ' ' . print_r($this->connection->getLastBody(), true));
        }
    }
}
```
Comme vous pouvez le constater, nous récupérons la configuration depuis le fichier ```config.yml``` et on lance la publication via la ressource ```statuses/update```. A noter, l'utilisation de la librairie ```abraham/twitteroauth``` pour gagner encore plus de temps et de ligne de code ;)

Il suffit maintenant de créer notre **SignalSlot** et de lancer la publication sur twitter ! Tout d'abord nous allons créer le fichier ```OnPublishSlot.php``` avec le contenu suivant:

```php
<?php

namespace Pvr\EzSocialBundle\Slot;

use eZ\Publish\API\Repository\ContentTypeService;
use eZ\Publish\Core\SignalSlot\Slot as BaseSlot;
use eZ\Publish\Core\SignalSlot\Signal;
use eZ\Publish\API\Repository\ContentService;
use Psr\Log\LoggerInterface;
use Pvr\EzSocialBundle\Networks\Handler\TwitterHandler;
use Pvr\EzSocialBundle\Networks\NetworkHandler;
use Pvr\EzSocialBundle\Networks\NetworkInterface;

class OnPublishSlot extends BaseSlot
{
    /**
     * @var \eZ\Publish\API\Repository\ContentService
     */
    private $contentService;
    /**
     * @var \eZ\Publish\API\Repository\ContentTypeService
     */
    private $contentTypeService;
    private $twitterService;
    private $logger;
    private $content_type;

    public function __construct(
        ContentService $contentService,
        ContentTypeService $contentTypeService,
        LoggerInterface $logger,
        NetworkInterface $twitterHandler,
        $content_type )
    {
        $this->contentService = $contentService;
        $this->logger = $logger;
        $this->twitterService = $twitterHandler;
        $this->content_type = $content_type;
        $this->contentTypeService = $contentTypeService;
    }

    /**
     * Receive the given $signal and react on it.
     *
     * @param Signal $signal
     */
    public function receive(Signal $signal)
    {
        if ( !$signal instanceof Signal\ContentService\PublishVersionSignal) {
            return;
        }

        // Load content
        $content = $this->contentService->loadContent( $signal->contentId, null, $signal->versionNo );
        $contentTypeIdentifier = $this->contentTypeService->loadContentType( $content->contentInfo->contentTypeId )->identifier;

        if (isset($this->content_type[$contentTypeIdentifier])) {
            $networks = $this->content_type[$contentTypeIdentifier]['network'];
            // Check all networks
            foreach ($networks as $network) {
                // If network exist ...
                if ($this->networkHandler->has($network)) {
                    $status     = $content->getFieldValue($this->content_type[$contentTypeIdentifier]['status'])->text;
                    $siteaccess = $this->content_type[$contentTypeIdentifier]['siteaccess'];
                    $locationId = $content->contentInfo->mainLocationId;
                    // @TODO: image

                    $handler = $this->networkHandler->get($network);
                    if ($handler == null) {
                        // throw exception
                    }
                    $handler->publish(['status' => $status, 'siteaccess' => $siteaccess, 'locationId' => $locationId]);
                }
            }
        }
    }
}
```

Le morceau de code ici est un peu plus complexe, car prévoit l'utilisation de plusieurs réseaux sociaux avec un filtrage sur le type de contenu. Par exemple il sera tout à fait possible de publier un ```article``` sur 2 comptes twitter différents, 1 facebook, etc.
La seule difficulté ici se trouve sur l'utilisation de l'objet ```signal``` mais avec un IDE comme phpStorm on retrouve très vite ces petits.
