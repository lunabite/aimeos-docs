If you want to perform actions on the basket content depending on current activity, basket plug-ins are the tool of choice. They allow you to operate on the whole basket and are able to add, remove or change products, services, coupons or addresses as you wish. Which basket plug-ins are available by default and how to configure them is described in the [user manual](../manual/plugins.md).

Your basket plugin must be part of your project specific [Aimeos extension](../developer/extensions.md) which you must create and then store at *./<yourext>/lib/custom/src/MShop/Plugin/Provider/Order/<classname>.php* to be available in your Aimeos installation.

For new basket plug-ins, you can use this skeleton class:

```php
namespace Aimeos\MShop\Plugin\Provider\Order;

class Myexample
    extends \Aimeos\MShop\Plugin\Provider\Factory\Base
    implements \Aimeos\MShop\Plugin\Provider\Iface, \Aimeos\MShop\Plugin\Provider\Factory\Iface
{
    private $singleton;

    public function register( \Aimeos\MW\Observer\Publisher\Iface $p )
    {
    }

    public function update( \Aimeos\MW\Observer\Publisher\Iface $basket, $event, $value = null )
    {
    }
}
```

The file containing the class must be placed in the *./lib/custom/src/MShop/Plugin/Provider/Order/Myexample.php* file (or anymore reasonable file/class name) of your extension. Here is an article about [creating extensions](../developer/extensions.md) for Aimeos, if you don't have one yet.

# Event system

The basket plug-in system is implemented as event driven process, meaning that your plug-in will get notified if something have happened. Plug-ins have to register for events they want to listen to, so only those plug-ins are executed that actually need to do something. They can listen to these events:

setCustomerId.before, setCustomerId.after
: Before and after the basket has been associated to the customer, plug-in receives the customer ID

setLocale.before, setLocale.after
: Before and after the locale item with site, language and currency has been added, plug-in receives the locale item

addProduct.before, addProduct.after
: Before and after the product item has been added, plug-in receives the order product item

deleteProduct.before, deleteProduct.after
: Before and after the product item has been deleted, plug-in receives the position of the product in the basket before and the deleted order product item afterwards

setAddress.before, setAddress.after
: Before and after the address item has been added, plug-in receives the order address item

deleteAddress.before, deleteAddress.after
: Before and after the address item has been deleted, plug-in receives the address type before (usually *\Aimeos\MShop\Order\Item\Base\Address\Base::TYPE_DELIVERY* or *\Aimeos\MShop\Order\Item\Base\Address\Base::TYPE_PAYMENT*) and the deleted order address item afterwards

setService.before, setService.after
: Before and after the delivery/payment service item has been added, plug-in receives the order service item

deleteService.before, deleteService.after
: Before and after the delivery/payment service item has been deleted, plug-in receives the service type before (usually *\Aimeos\MShop\Order\Item\Base\Service\Base::TYPE_DELIVERY* or *\Aimeos\MShop\Order\Item\Base\Service\Base::TYPE_PAYMENT*) and the deleted order service item afterwards

addCoupon.before, addCoupon.after
: Before and after the coupon has been added, plug-in receives the list of order product items that are associated to the code before and the coupon code afterwards

deleteCoupon.before, deleteCoupon.after
: Before and after the coupon has been deleted, plug-in receives the coupon code

check.before, check.after
: Before and after the basket content has been checked, plug-in receives the content types as bitmap (see [`PARTS_*` constants](https://github.com/aimeos/aimeos-core/blob/master/lib/mshoplib/src/MShop/Order/Item/Base/Base.php))

setOrder.before
: Before the order is stored in the database

To listen for such an event, your plug-in has to register itself at the publisher object which is the basket (or *\Aimeos\MShop\Order\Item\Base\Standard* to be more precise). This is done by calling the *addListener()* method of the publisher:

```php
public function register( \Aimeos\MW\Observer\Publisher\Iface $p )
{
    $p->addListener( $this, 'addProduct.after' );
    $p->addListener( $this, 'deleteProduct.after' );
    // ...
}
```

This would register the plug-in (the $this object) as listener for the *addProduct.after* and *deleteProduct.after* events. A plug-in can register itself to as many events as it needs but you should keep the list of events as short as possible because executing a long list of plug-ins on each action can take some time, especially if the plug-ins generate events itself by changing the content of the basket.

# Main code

The *update()* method of the plug-ins are executed by the basket if an event occurred the plug-in has registered itself to. Thus, you have to put the code that should manipulate the basket into this method. There are some things you should take care of:

Check passed order object
: It must implement *\Aimeos\MShop\Order\Item\Base\Iface*, otherwise you must throw an exception. This protects you against bugs in other extensions.

Protect against multiple invocations
: In event driven systems, your *update()* method can be called more than once if other plug-ins generate events too by changing the basket. If your code should only be executed once, set a singleton value in your class the first time and skip the code in subsequent calls.

Use plug-in configuration values
: The configuration consists of key/value pairs stored in an array. If a configuration value is required by the plug-in, you should test for it and handle a missing value by either throwing an exception or using a reasonable default value. The configuration options can be set in the [administration interface](../manual/plugin-details.md).

Throw exceptions
: If something goes wrong, throw an exception of the type *\Aimeos\MShop\Plugin\Provider\Exception*.This class has a special fourth parameter where specific information about the occurred problem can be passed. Please have a look at the [AddressAvailable](https://github.com/aimeos/aimeos-core/blob/master/lib/mshoplib/src/MShop/Plugin/Provider/Order/AddressesAvailable.php) and [ProductGone](https://github.com/aimeos/aimeos-core/blob/master/lib/mshoplib/src/MShop/Plugin/Provider/Order/ProductGone.php) plug-ins for more details.

Return a boolean value
: The method should return true if everything worked fine. If false is returned, the execution of all following plug-ins for this event is skipped. The advantage is that the code execution is not aborted completely by throwing an exception. You have control over the order of the executed plug-ins by the "position" property in the [administration interface](../manual/plugin-details.md). The plug-in with the lowest number is executed first. If two or more plug-ins share the same number, the order of these plug-ins is arbitrary.

The following implementation shows the important code blocks:

```php
public function update( \Aimeos\MW\Observer\Publisher\Iface $basket, $event, $value = null )
{
    $context = $this->getContext();
    $iface = '\Aimeos\MShop\Order\Item\Base\Iface';

    if( !( $basket instanceof $iface ) )
    {
        $msg = sprintf( 'Object is not of required type "%1$s"', $iface );
        throw new \Aimeos\MShop\Plugin\Provider\Exception( $msg );
    }

    if( $this->singleton === null )
    {
        $value = $this->getConfigValue( 'key', 'default' );

        // ...
        $this->singleton = true;
    }

    return true;
}
```

Your plug-in has access to the value handed over by the basket, which depends on the event (the event name is available in the $event parameter). Please have a look into the [list of events](#event-system) to find out what you can expect in $value.

!!! tip
    You can log plug-in activities using the "Log" decorator when you need to debug your code, so you will be able to retrace the actions in the log entries. Event-driven code sometimes tends to have surprising effects, especially if other plug-ins create more events by changing the basket content too.

# Decorators

They are a great way to add constraints to the basket plug-ins or to implement functionality that should be available for multiple plug-ins. Decorators are added in the administration interface by adding their name after the plug-in name, separated by a comma.

!!! note
    It is possible to apply decorators to basket plug-ins globally using the [mshop/plugin/provider/order/decorators](../config/mshop/plugin-provider.md#decorators) configuration setting. Named decorators listed in this configuration array are applied to all plug-ins.

All you need to do is to extend from the *\Aimeos\MShop\Plugin\Provider\Decorator\Base* class and overwrite *update()* method where you can apply additional rules to the execution of the original method. Here's an example skeleton for a plug-in decorator:

```php
namespace \Aimeos\MShop\Plugin\Provider\Decorator;

class Example
    extends \Aimeos\MShop\Plugin\Provider\Decorator\Base
    implements \Aimeos\MShop\Plugin\Provider\Decorator\Iface
{
    public function update( \Aimeos\MW\Observer\Publisher\Iface $order, $action, $value = null )
    {
        if( <condition> ) {
            return $this->getProvider()->update( $order, $action, $value );
        }

        return true;
    }
}
```

The advantage of this approach is, that multiple decorators can be used for one plug-in and that one decorator can be used by multiple plug-ins. This way, the common rules are available for all plug-ins and you can add or remove those rules dynamically without touching the code of your plug-ins.

These rules can be for example:

* execute only for certain locales
* execute only for registered and logged in customers who already ordered multiple times
* execute only when the products in the basket exceed a certain amount of money
* execute only when the amount of products in the basket exceeds a certain threshold
* and everything else you can image and express in PHP ...

# Testing

Plug-ins and their decorators are objects which can be tested very good but the amount of code required corresponds to the number of managers used inside the classes. As plug-ins mainly operate on a basket instance, unit tests only need to check if the basket content changed in the expected way.

A test skeleton for a plug-in or decorator is:

```php
namespace \Aimeos\MShop\Plugin\Provider\Order;

class ExampleTest extends \PHPUnit_Framework_TestCase
{
    private $object;
    private $basket;

    protected function setUp()
    {
        $context = TestHelperMShop::getContext();

        $pluginManager = \Aimeos\MShopFactory::createManager( $context, 'plugin' );
        $orderManager = \Aimeos\MShop\Factory::createManager( $context, 'order' );
        $orderBaseManager = $orderManager->getSubManager( 'base' );

        $this->basket = $orderBaseManager->createItem();
        $plugin = $pluginManager->createItem();

        $this->object = new \Aimeos\MShop\Plugin\Provider\Order\Example( $context, $plugin );
    }

    public function testRegister()
    {
        $this->object->register( $this->basket );
    }

    public function testUpdate()
    {
        $this->assertTrue( $object->update( $this->basket, 'check.after' ) );
    }
}
```

You should implement more tests for the *update()* method until every line inside is executed at least once. For more information regarding unit tests please have a look into the [PHPUnit documentation](https://phpunit.readthedocs.io/en/latest/writing-tests-for-phpunit.html). The chapter about [stubs and mocks](https://phpunit.readthedocs.io/en/latest/test-doubles.html) is especially useful if you want to replace the manager objects used in your plug-in during the tests by injecting a mock object into the Aimeos manager factories via the [*inject()*](https://github.com/aimeos/aimeos-core/blob/master/lib/mshoplib/src/MShop.php) method.
