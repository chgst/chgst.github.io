---
layout: default
title: Symfony bundle
permalink: /symfony-bundle/
---

# Symfony bundle

If you work with Symfony and you would like to implement CQRS/ES in your project, there is a bundle which provides
all the configuration you need to get it running. Here is a step by step guide.

I'm going to create very basic CRUD application based on Doctrine and SQLite. **After** that, I'm going to add **Changeset**
and implement write model, projector and listener.

### Start new Symfony project

```bash
symfony new chgst-bundle-example 3.4
```

### Create read model PHP class

```php
<?php

# src/AppBundle/Entity/Building.php

namespace AppBundle\Entity;

class Building
{
    /** @var string */
    private $id;

    /** @var array */
    private $persons = [];

    public function getId()
    {
        return $this->id;
    }

    public function setId($id)
    {
        $this->id = $id;

        return $this;
    }

    public function getPersons()
    {
        return $this->persons;
    }

    public function addPerson($person)
    {
        if( ! in_array($person, $this->persons))
        {
            $this->persons[] = $person;
        }

        return $this;
    }

    public function removePerson($person)
    {
        if (($key = array_search($person, $this->persons)) !== false)
        {
            unset($this->persons[$key]);
        }

        return $this;
    }
}
```

Create Doctrine mapping

```bash
mkdir -p src/AppBundle/Resources/config/doctrine && \
touch src/AppBundle/Resources/config/doctrine/Building.orm.yml
```

Create YAML mapping:

```yaml
AppBundle\Entity\Building:
  type: entity
  id:
    id:
      type: string
      id: true
      generator:
        strategy: none
  fields:
    persons:
      type: string
      length: 4000
```

Change Doctrine configuration to use SQLite in config.yml

```yaml
# app/config/config.yml

doctrine:
    dbal:
        driver:   pdo_sqlite
        path:     "%kernel.root_dir%/../var/data.db3"
```

and run doctrine schema create command

```bash
php bin/console doctrine:database:create && \
php bin/console doctrine:schema:create
```


```bash
composer require chgst/chgst-bundle
```
