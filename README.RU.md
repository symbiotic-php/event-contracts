## Расширение интерфейсов для диспетчера событий PSR-14

Добавлен метод добавления слушателей с передачей имени события и возможностью передавать в качестве слушателя название класса.
Такой подход позволяет избежать использования Рефлексии при добавлении слушателей, а также ненужных объектов замыканий.

## Установка
```
composer require symbiotic/event-contracts

Можно сразу поставить реализацию интерфейсов "symbiotic/event"
```
## Описание
```php
use \Psr\EventDispatcher\ListenerProviderInterface;

interface ListenersInterface extends ListenerProviderInterface
{
    /**
     * @param string $event имя класса или произвольное имя события
     * (при произвольного названии нужен кастомный диспетчер не по PSR)
     *
     * @param \Closure|string $handler функция или имя класса обработчика
     * Класс обработчика событий должен реализовать метод handle (...$params) или __invoke(...$params)
     * <Важно:> При добавлении слушателей в качестве названий  классов, вы должны будете их при отдаче в методе getListenersForEvent() адаптировать в \Closure!!!
     *
     * @return void
     */
    public function add(string $event, $handler): void;
}
```

Примерная реализация \Closure обертки:
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
// вы можете реализовать свою обертку прямо в методе getListenersForEvent() или прокинуть резолвер с контейнером PSR

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
 *  Прокидывание обертки
 */
$listenersProvider = new \Symbiotic\Event\ListenerProvider($listenerResolver);
```

