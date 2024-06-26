<h1 align="center">
Loggastic<br>
    <a href="https://packagist.org/packages/locastic/loggastic" title="License" target="_blank">
        <img src="https://img.shields.io/packagist/l/locastic/loggastic.svg" />
    </a>
    <a href="https://packagist.org/packages/locastic/loggastic" title="Version" target="_blank">
        <img src="https://img.shields.io/packagist/v/Locastic/loggastic.svg" />
    </a>
    <a href="https://scrutinizer-ci.com/g/Locastic/Loggastic/" title="Scrutinizer" target="_blank">
        <img src="https://img.shields.io/scrutinizer/g/Locastic/Loggastic" />
    </a>
    <a href="https://packagist.org/packages/locastic/loggastic" title="Total Downloads" target="_blank">
        <img src="https://poser.pugx.org/locastic/loggastic/downloads" />
    </a>
</h1>

Loggastic is made for tracking changes to your objects and their relations.
Built on top of the **Symfony framework**, this library makes it easy to implement activity logs and store them on **Elasticsearch** for fast logs browsing.

Each tracked entity will have two indexes in the ElasticSearch:
1. `entity_name_activity_log` -> saving all CRUD actions made on an object. And additionally saving before and after values for Edit actions.
2. `entity_name_current_data_tracker` -> saving the latest object values used for comparing the changes made on Edit actions. This enables us to only store before and after values for modified fields in the `activity_log` index

System requirements
-------------------

Elasticsearch version 7.17

Installation
------------

`composer require locastic/loggastic`

Making your entity loggable
---------------------------

To make your entity loggable you need to do the following steps:
### 1. Add Loggable attribute to your entity

Add `Locastic\Loggastic\Annotation\Loggable` annotation to your entity and define serialization group name:
```php
<?php

namespace App\Entity;

use Locastic\Loggastic\Annotation\Loggable;

#[Loggable(groups: ['blog_post_log'])]
class BlogPost
{    
    // ...
}
```

If you are using YAML:
```yaml
locastic_loggable:
        - { class: 'App\Entity\BlogPost', groups: [ 'blog_post_log' ] }
```

Or XML:
```xml
<?xml version="1.0" ?>

<locastic_loggable_classes xmlns="https://locastic.com/schema/metadata/loggable"
                           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                           xsi:schemaLocation="https://locastic.com/schema/metadata/loggable
           https://locastic.com/schema/metadata/loggable.xsd" >
    <loggable_class class="App\Entity\BlogPost">
        <group name="blog_post_log"/>
    </loggable_class>
</locastic_loggable_classes>
```

### 2. Add serialization groups to the fields you want to log
Use the serialization group defined in the Loggable attribute config on the fields you want to track.
You can add them to the relations and their fields too.
```php
<?php

namespace App\Entity;

use Locastic\Loggastic\Annotation\Loggable;
use Symfony\Component\Serializer\Annotation\Groups;

#[Loggable(groups: ['blog_post_log'])]
class BlogPost
{
    private int $id;

    #[Groups(groups: ['blog_post_log'])]
    private string $title;

    #[Groups(groups: ['blog_post_log'])]
    private ArrayCollection $tags;
    
    // ...
}
```

Example for logging fields from relations:

```php
<?php

namespace App\Entity;

use Locastic\Loggastic\Annotation\Loggable;
use Symfony\Component\Serializer\Annotation\Groups;

class Tag
{
    private int $id;

    #[Groups(groups: ['blog_post_log'])]
    private string $name;

    #[Groups(groups: ['blog_post_log'])]
    private DateTimeImmutable $createdAt;
    
    // ...
}
```

Note: You can also use **annotations, xml and yaml**! Examples coming soon.

### 3. Run commands for creating indexes in ElasticSearch

    bin/console locastic:activity-logs:create-loggable-indexes

If you already have some data in the database, make sure to populate current data trackers with the following command:

    bin/console locastic:activity-logs:populate-current-data-trackers

### 4. Displaying activity logs
Here are the examples for displaying activity logs in twig or as Api endpoints:

**a) Displaying logs in Twig**
`Locastic\Loggastic\DataProvider\ActivityLogProviderInterface` service comes with a few useful methods for getting the activity logs data:
```php
    public function getActivityLogsByClass(string $className, array $sort = []): array;

    public function getActivityLogsByClassAndId(string $className, $objectId, array $sort = []): array;

    public function getActivityLogsByIndexAndId(string $index, $objectId, array $sort = []): array;
```
Use them to fetch data from the Elasticsearch and display it in your views. Example for displaying results in Twig:
```twig
Activity logs for Blog Posts:
<br>
{% for log in activityLogs %}
    {{ log.action }} {{ log.objectType }} with {{ log.objectId }} ID at {{ log.loggedAt|date('d.m.Y H:i:s') }} by {{ log.user.username }}
{% endfor %}
```
The output would look something like this:
```text
Activity logs for Blog Posts:

Created BlogPost with 1 ID at 01.01.2023 12:00:00 by admin
Updated BlogPost with 1 ID at 02.01.2023 08:30:00 by admin
Deleted BlogPost with 1 ID at 01.01.2023 12:00:00 by admin
```

**b) Displaying logs in the api endpoint using ApiPlatform**
In order to display Loggastic activity logs in an ApiPlatform endpoint, you can use ApiPlatforms ElasticSearch integration: https://api-platform.com/docs/core/elasticsearch/

Example for displaying activity logs in the ApiPlatform endpoint:
```php
#[ApiResource(
    operations: [
        new Get(provider: ItemProvider::class),
        new GetCollection(provider: CollectionProvider::class),
    ],
    order: ["loggedAt" => "DESC"],
    stateOptions: new Options(index: '*_activity_log'),
)]
class ActivityLog extends BaseActivityLog
{
    #[ApiProperty(identifier: true)]
    protected ?string $id = null;
}
```

You can easily filter the results using the existing ApiPlatform filters: https://api-platform.com/docs/core/filters/. 
If you want to have different fields in the response, use serialization groups or even create a custom DTO.

Using `*_activity_log` index will return all activity logs. 
If you want to return only logs for one entity, use the exact index name. For example if you only want to show `BlogPost` entity logs, use `blog_post_activity_log` index in `stateOptions` config.


That's it!
----------
Now you have the basic activity logs setup. 
Each time some change happens in the database for loggable entities, the activity log will be saved to the Elasticsearch.

## Customization guide
Now that you have the basic setup, you can add some additional options and customize the library to your needs.

## Configuration reference
Default configuration:
```yaml
# config/packages/loggastic.yaml
locastic_loggastic:
    # directory paths containing loggable classes or xml/yaml files
    loggable_paths:
        - '%kernel.project_dir%/Resources/config/loggastic'
        - '%kernel.project_dir%/src/Entity'

    # Turn on/off the default Doctrine subscriber
    default_doctrine_subscriber: true

    # Turn on/off collection identifier extractor 
    # if set to `true` objects identifiers in collections will be used as array keys
    # if set to `false` default numeric array keys will be used
    identifier_extractor: true
    
    # ElasticSearch config
    elastic_host: 'localhost:9200'
    elastic_date_detection: true    #https://www.elastic.co/guide/en/elasticsearch/reference/current/date-detection.html
    elastic_dynamic_date_formats: "strict_date_optional_time||epoch_millis||strict_time"

    # ElasticSearch index mapping for ActivityLog. https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html#mappings
    activity_log:
        elastic_properties:
            id:
                type: keyword
            action:
                type: text
            loggedAt:
                type: date
            objectId:
                type: text
            objectType:
                type: text
            objectClass:
                type: text
            dataChanges:
                type: text
            user:
                type: object
                properties:
                    username:
                        type: text

    # ElasticSearch index mapping for CurrentDataTracker
    current_data_tracker:
        elastic_properties:
            dateTime:
                type: date
            objectId:
                type: text
            objectType:
                type: text
            objectClass:
                type: text
            data:
                type: text

```

### Saving logs async
Activity logs are using Symfony messenger component and are made to work in the async way too.
If you want to make them async add the following messages to the messenger config:
```yaml
framework:
    messenger:
        routing:
            'Locastic\Loggastic\Message\PopulateCurrentDataTrackersMessage': async
            'Locastic\Loggastic\Message\CreateActivityLogMessage': async
            'Locastic\Loggastic\Message\DeleteActivityLogMessage': async
            'Locastic\Loggastic\Message\UpdateActivityLogMessage': async
```

***Important note!***

Only one consumer should be used per loggable object in order to not corrupt the data.

### Optimising messenger for large amount of data
If you have a large amount of data, you might need more than one consumer to process the messages.
In that case, you can configure different transports for the messages and use different consumer for each one.
First step is to configure the transports. Here are the examples for AMQP and Doctrine transports for the `activity_logs_default` and `activity_logs_product` queues:

AMQP transport config example:
```yaml
framework:
    messenger:
        transports:
             activity_logs_default:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    exchange:
                        name: activity_logs_default
                    queues:
                        activity_logs_default: ~
             activity_logs_product:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queues:
                        activity_logs_product: ~
                    exchange:
                        name: activity_logs_product

        routing:
            'Locastic\Loggastic\Message\PopulateCurrentDataTrackersMessage': activity_logs_default
            'Locastic\Loggastic\Message\CreateActivityLogMessage': activity_logs_default
            'Locastic\Loggastic\Message\DeleteActivityLogMessage': activity_logs_default
            'Locastic\Loggastic\Message\UpdateActivityLogMessage': activity_logs_default
```

Doctrine transport config example:
```yaml
framework:
    messenger:
        transports:
             activity_logs_default:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: activity_logs_default
             activity_logs_product:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queue_name: activity_logs_product
        routing:
            'Locastic\Loggastic\Message\PopulateCurrentDataTrackersMessage': activity_logs_default
            'Locastic\Loggastic\Message\CreateActivityLogMessage': activity_logs_default
            'Locastic\Loggastic\Message\DeleteActivityLogMessage': activity_logs_default
            'Locastic\Loggastic\Message\UpdateActivityLogMessage': activity_logs_default
```

Next step is to decorate `ActivityLogDispatcher` and add your own logic for dispatching messages to the transports.
In this example we are sending all messages to the `activity_logs_default` transport except the ones for the `Product` entity which are sent to the `activity_logs_product` transport:

```php
<?php

namespace App\MessageDispatcher;

use App\Entity\Product;
use Locastic\Loggastic\Message\ActivityLogMessageInterface;
use Locastic\Loggastic\MessageDispatcher\ActivityLogMessageDispatcherInterface;
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;

#[AsDecorator(ActivityLogMessageDispatcherInterface::class)]
class ActivityLogMessageDispatcher implements ActivityLogMessageDispatcherInterface
{
    public function __construct(private readonly ActivityLogMessageDispatcherInterface $decorated)
    {
    }

    public function dispatch(ActivityLogMessageInterface $activityLogMessage, ?string $transportName = null): void
    {
        if ($activityLogMessage->getClassName() === Product::class) {
            $this->decorated->dispatch($activityLogMessage, 'activity_logs_product');

            return;
        }

        $this->decorated->dispatch($activityLogMessage, $transportName);
    }
}
```

Depending on your project needs, you can have more transports and dispatch messages to them based on your own logic.

### Handling relations
Sometimes you want to log changes made on some entity to some related entity. For example if you are using the Doctrine listener, you will only get the entity that actually had changes.
Let's say you want to log `Product` changes which has a relation to the `ProductVariant`. On the edit form only fields from the `ProductVariant` were changed.
Even if you run `persist()` method on `Product`, in this case only `ProductVariant` will be shown in the Doctrine listener.
For this case you can use the `Locastic\Loggastic\Loggable\LoggableChildInterface` on `ProductVariant`:
```php
<?php

namespace App\Entity;

use Locastic\Loggastic\Loggable\LoggableChildInterface;

class ProductVariant implements LoggableChildInterface
{
    private Product $product;
    
    public function getProduct(): Product
    {
        return $this->product;
    }
    
    public function logTo(): ?object
    {
        return $this->getProduct();
    }
    
    // ...
}
```

Now each change made on `ProductVariant` will be logged to the `Product`.

### Custom event listeners for saving activity logs
You can use `Locastic\Loggastic\Logger\ActivityLoggerInterface` service to save item changes to the Elasticsearch:
```php
<?php
namespace App\Service;

class SomeService
{
    public function __construct(private readonly ActivityLoggerInterface $activityLogger)
    {
    }
    
    public function logItem($item): void
    {
        $this->activityLogger->logCreatedItem($item, 'custom_action_name');
        $this->activityLogger->logDeletedItem($item->getId(), get_class($item), 'custom_action_name');
        $this->activityLogger->logUpdatedItem($item, 'custom_action_name');
    }
}
```
Depending on you application logic, you need to find the most fitting place to trigger logs saving.

In most cases that can be the ***Doctrine event listener*** which is triggered on each database change. Loggastic comes with a built-in Doctrine listener which is used by default.
If you want to turn it off, you can do it by setting the `loggastic.doctrine_listener_enabled` config parameter to `false`:
```yaml
# config/packages/loggastic.yaml

loggastic:
    doctrine_listener_enabled: false
```

If you are using ***ApiPlatform***, one of the good options would be to use its POST_WRITE event: https://api-platform.com/docs/core/events/#custom-event-listeners

And for the ***Sylius projects*** you can use the Resource bundle events: https://docs.sylius.com/en/1.12/book/architecture/events.html

### Save activity logs when no data changes were made
Sometimes you want to save activity logs even if no data changes were made. 
For example if you want to log order confirmation email was sent or some PDF was downloaded.

You can do that by setting the 3rd parameter to true:
```php
$this->activityLogger->logUpdatedItem($item, 'Order confirmation sent', true);
```

## Contribution

If you have idea on how to improve this bundle, feel free to contribute. If you have problems or you found some bugs, please open an issue.

## Support

Want us to help you with this bundle or any ApiPlatform/Symfony project? Write us an email on info@locastic.com
