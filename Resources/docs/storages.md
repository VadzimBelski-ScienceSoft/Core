# Storages

Storage allow you save,fetch payment related information. 
They could be used explicitly, it means you have to call save or fetch methods when it is required. 
Or you can integrate a storage to a payment using `StorageExtension`. 
In this case every time payment finish to execute a request it stores the information. 
`StorageExtension` could also load a model by it is `Identificator` so you do not have to care about that.

Explicitly used example:

```php
<?php
use Payum\Core\Storage\FilesystemStorage;

$storage = new FilesystemStorage('/path/to/storage', 'Payum\Core\Model\Order', 'number');

$order = $storage->create();
$order->setTotalAmount(123);
$order->setCurrency('EUR');

$storage->update($order);

$foundOrder = $storage->find($order->getNumber());
```

Implicitly used example: 

```php
<?php
use Payum\Core\Extension\StorageExtension;
use Payum\Core\Payment;
use Payum\Core\Storage\FilesystemStorage;

$payment->addExtension(new StorageExtension(
   new FilesystemStorage('/path/to/storage', 'Payum\Core\Model\Order', 'number')
));
```

Usage of a model identity with the extension:

```php
<?php
use Payum\Core\Extension\StorageExtension;
use Payum\Core\Model\Identity;
use Payum\Core\Storage\FilesystemStorage;
use Payum\Core\Payment;
use Payum\Core\Request\Capture;

$storage = new FilesystemStorage('/path/to/storage', 'Payum\Core\Model\Order', 'number');

$order = $storage->create();
$storage->update($order);

$payment->addExtension(new StorageExtension($storage));

$payment->execute($capture = new Capture(
    $storage->identify($order)
));

echo get_class($capture->getModel());
// -> Payum\Core\Model\Order
```

## Doctrine ORM

```
php composer.phar install "doctrine/orm"
```

Add token and order classes:

```php
<?php
namespace Acme\Entity;

use Doctrine\ORM\Mapping as ORM;
use Payum\Core\Model\Token;

/**
 * @ORM\Table
 * @ORM\Entity
 */
class PaymentToken extends Token
{
}
```

```php
<?php
namespace Acme\Entity;

use Doctrine\ORM\Mapping as ORM;
use Payum\Core\Model\Order as BaseOrder;

/**
 * @ORM\Table
 * @ORM\Entity
 */
class Order extends BaseOrder
{
    /**
     * @ORM\Column(name="id", type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="IDENTITY")
     *
     * @var integer $id
     */
    protected $id;
}
```

next, you have to create an entity manager and Payum's storage:

```php
<?php
use Doctrine\Common\Cache\ArrayCache;
use Doctrine\Common\Persistence\Mapping\Driver\MappingDriverChain;
use Doctrine\ORM\Configuration;
use Doctrine\ORM\EntityManager;
use Doctrine\ORM\Mapping\Driver\SimplifiedXmlDriver;
use Payum\Core\Bridge\Doctrine\Storage\DoctrineStorage;

$config = new Configuration();
$driver = new MappingDriverChain;

// payum's basic models
$driver->addDriver(
    new SimplifiedXmlDriver(array('path/to/Payum/Core/Model' => 'Payum\Core\Model')), 
    'Payum\Core\Model'
);

// your models
$driver->addDriver(
    $config->newDefaultAnnotationDriver(array('path/to/Acme/Entity'), false), 
    'Acme\Entity'
);

$config->setAutoGenerateProxyClasses(true);
$config->setProxyDir(\sys_get_temp_dir());
$config->setProxyNamespace('Proxies');
$config->setMetadataDriverImpl($driver);
$config->setQueryCacheImpl(new ArrayCache());
$config->setMetadataCacheImpl(new ArrayCache());

$connection = array('driver' => 'pdo_sqlite', 'path' => ':memory:');

$orderStorage = new DoctrineStorage(
   EntityManager::create($connection, $config),
   'Payum\Entity\Order'
);

$tokenStorage = new DoctrineStorage(
   EntityManager::create($connection, $config),
   'Payum\Entity\PaymentToken'
);
```

### Doctrine MongoODM.

```
php composer.phar install "doctrine/mongodb": "1.0.*@dev" "doctrine/mongodb-odm": "1.0.*@dev"
```

```php
<?php
namespace Acme\Document;

use Doctrine\ODM\MongoDB\Mapping\Annotations as Mongo;
use Payum\Core\Model\Token;

/**
 * @Mongo\Document
 */
class PaymentToken extends Token
{
}
```

```php
<?php
namespace Acme\Document;

use Doctrine\ODM\MongoDB\Mapping\Annotations as Mongo;
use Payum\Core\Model\Order as BaseOrder;

/**
 * @Mongo\Document
 */
class Order extends BaseOrder
{
    /**
     * @Mongo\Id
     *
     * @var integer $id
     */
    protected $id;
}
```

next, you have to create an entity manager and Payum's storage:

```php
<?php
use Doctrine\Common\Annotations\AnnotationReader;
use Doctrine\Common\Cache\ArrayCache;
use Doctrine\Common\Persistence\Mapping\Driver\MappingDriverChain;
use Doctrine\Common\Persistence\Mapping\Driver\MappingDriver;
use Doctrine\Common\Persistence\Mapping\Driver\SymfonyFileLocator;
use Doctrine\ODM\MongoDB\Types\Type;
use Doctrine\ODM\MongoDB\Mapping\Driver\AnnotationDriver;
use Doctrine\ODM\MongoDB\Mapping\Driver\XmlDriver;
use Doctrine\ODM\MongoDB\DocumentManager;
use Doctrine\ODM\MongoDB\Configuration;
use Doctrine\MongoDB\Connection;

Type::addType('object', 'Payum\Core\Bridge\Doctrine\Types\ObjectType');

$driver = new MappingDriverChain;

// payum's basic models
$driver->addDriver(
    new XmlDriver(
       new SymfonyFileLocator(array(
            '/path/to/Payum/Core/Bridge/Doctrine/Resources/mapping' => 'Payum\Core\Model'
        ), '.mongodb.xml'),
        '.mongodb.xml'
    ), 
    'Payum\Core\Model'
);

// your models
AnnotationDriver::registerAnnotationClasses();
$driver->addDriver(
    new AnnotationDriver(new AnnotationReader(), array(
        'path/to/Acme/Document',
    )), 
    'Acme\Document'
);

$config = new Configuration();
$config->setProxyDir(\sys_get_temp_dir());
$config->setProxyNamespace('Proxies');
$config->setHydratorDir(\sys_get_temp_dir());
$config->setHydratorNamespace('Hydrators');
$config->setMetadataDriverImpl($driver);
$config->setMetadataCacheImpl(new ArrayCache());
$config->setDefaultDB('payum_tests');

$connection = new Connection(null, array(), $config);

$orderStorage = new DoctrineStorage(
    DocumentManager::create($connection, $config),
    'Acme\Document\Order'
);

$tokenStorage = new DoctrineStorage(
    DocumentManager::create($connection, $config),
    'Acme\Document\SecurityToken'
);
```        

## Filesystem.

```php
<?php
use Payum\Core\Storage\FilesystemStorage;

$storage = new FilesystemStorage(
    '/path/to/storage', 
    'Payum\Core\Model\Order', 
    'number'
);
```

## Custom.

You can create your own custom storage. To do so just implement `StorageInterface`.

```php
<?php
use Payum\Core\Storage\StorageInterface;

class CustomStorage implements StorageInterface
{
    // implement all declared methods
}
```

## TODO

* [Propel](http://propelorm.org/) Storage - https://github.com/Payum/Payum/pull/144
* [Pdo](http://php.net/manual/en/book.pdo.php) Storage - https://github.com/Payum/Payum/issues/205
* [Eloquent](http://laravel.com/docs/4.2/eloquent) Storage
* [Yii ActiveRecord](http://www.yiiframework.com/doc/guide/1.1/en/database.ar) Storage - https://github.com/Payum/PayumYiiExtension/pull/4


## Next Step

* [Get it started](get-it-started.md).
* [Back to index](index.md).
