## Extend interfaces for PSR-14 Event Dispatcher
README.RU.md  [РУССКОЕ ОПИСАНИЕ](https://github.com/symbiotic-php/event-contracts/blob/master/README.RU.md)

Added a method for adding listeners with passing the event name and the ability to pass the class name as a listener.
This approach avoids the use of Reflection when adding listeners, as well as unnecessary closure objects.

## Installation
```
composer require symbiotic/event-contracts

suggest symbiotic/event - reailsation 
```
## Description
```php
use \Psr\EventDispatcher\ListenerProviderInterface;

interface ListenersInterface extends ListenerProviderInterface
{
    /**
     * @param string $event the class name or an arbitrary event name
     * (with an arbitrary name, you need a custom dispatcher not for PSR)
     *
     * @param \Closure|string $handler function or class name of the handler
     * The event handler class must implement the handle method  (...$params) or __invoke(...$params)
     * Important! When adding listeners as class names, you will need to adapt them to \Closure
     * when you return them in the getListenersForEvent() method!!!
     *
     * @return void
     */
    public function add(string $event, $handler): void;
}
```

Sample implementation of the \Closure wrapper:
```php

$listenerResolver = function($listener) {
    return function(object $event) use ($listener) {
            // if classname
            if(is_string($listener) && class_exists($listener)) {
                $listener = new $listener();
                return $listener->handle($event);
            } elseif(is_callable($listener)) {
                return $listener($event);
            }
    };
};
// You can implement your wrapper directly in the getListenersForEvent() method or throw a resolver with a PSR container

class ListenerProvider implements Symbiotic\Event\ListenersInterface
{
    protected $listenerResolver;

    protected $listeners = [];

    public function __construct(\Closure $listenerResolver = null)
    {
        $this->listenerResolver = $listenerResolver;
    }

    public function add(string $event, $handler): void
    {
        $this->listeners[$event][] = $handler;
    }

    public function getListenersForEvent(object $event): iterable
    {
        $parents = \class_parents($event);
        $implements = \class_implements($event);
        $classes = array_merge([\get_class($event)], $parents ?: [], $implements ?: []);
        $listeners = [];
        foreach ($classes as $v) {
            $listeners = array_merge($listeners, isset($this->listeners[$v]) ? $this->listeners[$v] : []);
        }
        $wrapper = $this->listenerResolver;

        return $wrapper ? array_map(function ($v) use ($wrapper) {
            return $wrapper($v);
        }, $listeners) : $listeners;

    }
}

/**
 *  use resolver
 **/
$listenersProvider = new \Symbiotic\Event\ListenerProvider($listenerResolver);
```