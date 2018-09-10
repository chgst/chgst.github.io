---
layout: page
title: Documentation
permalink: /documentation/
---

# Installation

```bash
composer require chgst/chgst
```

If you want to use doctrine as persistence run this command

```bash
composer require chgst/persistence-docrtine doctrine/orm
``` 

# Usage

Create a bootstrap file where we add some configuration

```php
<?php
# bootstrap.php

use Doctrine\ORM\Tools\Setup;
use Doctrine\ORM\EntityManager;

require_once './vendor/autoload.php';

$config = Setup::createYAMLMetadataConfiguration(array(__DIR__."/config/yaml"), true);

$conn = array(
    'driver' => 'pdo_sqlite',
    'path' => __DIR__ . '/db.sqlite',
);

$entityManager = EntityManager::create($conn, $config);
```

Now create configuration folder and Yaml file with doctrine schema for Event model.

```bash
mkdir -p config/yaml && touch config/yaml/DefaultEvent.dcm.yml
```

Open `DefaultEvent.dcm.yml` in editor and write the mapping information

```yaml
# config/yaml/DefaultEvent.dcm.yml
DefaultEvent:

  type: entity

  table: event_stream

  id:
    id:
      type: integer
      generator:
        strategy: AUTO

  fields:
    name:
      type: string
    aggregateId:
      type: string
    aggregateType:
      type: string
    payload:
      type: string

```

Now create PHP class which represents event described above

```php
<?php
# src/DefaultEvent.php

class DefaultEvent extends Changeset\Event\Event
{
    protected $id;

    public function getId()
    {
        return $this->id;
    }

    public function setId($id)
    {
        $this->id = $id;

        return $this;
    }
}
```

Create schema mapping in the database:

```bash
./vendor/bin/doctrine orm:schema-tool:update -f
```

Create new file called `example.php` with following content.

```php
<?php
# example.php

use Changeset\Command\Command;
use Changeset\Command\Handler;
use Changeset\Communication\InMemoryCommandBus;
use Changeset\Communication\InMemoryEventBus;
use Changeset\Event\ObjectRepository;
use Symfony\Component\EventDispatcher\EventDispatcher;

require_once 'bootstrap.php';

$repository = new ObjectRepository($entityManager, DefaultEvent::class);
$dispatcher = new EventDispatcher();

$handler = new Handler($dispatcher, $repository);

$eventBus = new InMemoryEventBus();

$commandBus = new InMemoryCommandBus();
$commandBus->setHandler($handler);
$commandBus->setEventBus($eventBus);

$commandBus->dispatch(new Command('enter_building', 'Building', 'main', json_encode(['user' => 'bob'])));
```

You can now run the code `php example.php` and inspect database table `event_stream`. You should see that this has
been persisted in database.

The code above is enough to save data in the write model. It does not project data to read models or calls any listeners
yet. Lets call a listener which will print event to console output.

First let's create a listener class:

```php
<?php
# src/ConsoleOutListener.php

use Changeset\Event\EventInterface;
use Changeset\Event\ListenerInterface;

class ConsoleOutListener implements ListenerInterface
{
    public function supports(EventInterface $event): bool
    {
        return true; // let's support every event
    }

    public function process(EventInterface $event)
    {
        printf(
            "An event named: \"%s\" occured on %s with id \"%s\" with payload %s\n",
            $event->getName(), $event->getAggregateType(), $event->getAggregateId(), $event->getPayload()
        );
    }
}
```

Now add these 3 lines just above `$commandBus->dispatch(...)`:

```php
<?php

// other code here

$consoleOut = new ConsoleOutListener();

$eventBus->addListener($consoleOut);
$eventBus->enableListeners();

$commandBus->dispatch(new Command('enter_building', 'Building', 'main', json_encode(['user' => 'bob'])));
```

Now run the code and you shall see the message in the console:

```
An event named: "enter_building_completed" occured on Building with id "main": {"user":"bob"}
```

Now I'm going to add projector which will persist information into read model. Let's start with creating data read model for
Building. Create PHP class:

```php
<?php

class Building
{
    /** @var string */
    public $id;

    /** @var array */
    public $people;
}
```

I have user public properties here to make code shorter, but please do not do it, use getters and setters instead.

Now create Doctrine mapping

```yaml
# config/yaml/Building.dcm.yml
Building:

  type: entity

  id:
    id:
      type: string
      strategy: none

  fields:
    people:
      type: string

```

Update schema mapping in database:

```bash
./vendor/bin/doctrine orm:schema-tool:update -f
```

Now, create projector PHP class:

```php
<?php

# src/BuildingEnterProjector.php

use Changeset\Event\EventInterface;
use Changeset\Event\ProjectorInterface;
use Doctrine\ORM\EntityManager;

class BuildingEnterProjector implements ProjectorInterface
{
    /** @var EntityManager */
    private $em;

    public function __construct(EntityManager $em)
    {
        $this->em = $em;
    }

    public function supports(EventInterface $event): bool
    {
        return $event->getName() == 'enter_building_completed';
    }

    public function process(EventInterface $event)
    {
        $building = $this->em->getRepository(Building::class)->find($event->getAggregateId());

        if( ! $building)
        {
            $building = new Building();
            $building->id = $event->getAggregateId();
            $building->people = '[]';
        }

        $people = json_decode($building->people, true);
        $payload = json_decode($event->getPayload(), true);
        
        if( ! in_array($payload['user'], $people))
        {
            $people[] = $payload['user'];
            $building->people = json_encode($people);

            $this->em->persist($building);
            $this->em->flush($building);
        }

        printf(
            "There are currently %d person(s) in the %s %s\n",
            count($people), $event->getAggregateId(), $event->getAggregateType()
        );
    }
}

```

And add projector to event bus:

```php
<?php

// other code here

$buildingEnterProjector = new BuildingEnterProjector($entityManager);
$eventBus->addProjector($buildingEnterProjector);

$commandBus->dispatch(new Command('enter_building', 'Building', 'main', json_encode(['user' => 'bob'])));
```

Now when you run it again with `php example.php` you should see:

```
There are currently 1 person(s) in the main Building
An event named: "enter_building_completed" occured on Building with id "main" with payload {"user":"bob"}
```

This is it!