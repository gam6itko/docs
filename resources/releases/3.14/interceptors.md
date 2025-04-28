## New Interceptors

The `HMVC` package has been deprecated. It has been replaced by the new `spiral/interceptors` package, where we have reworked the interceptors. The basic principle remains the same, but the interface is now more understandable and convenient.

### InterceptorInterface

In the old `CoreInterceptorInterface`, the `$controller` and `$action` parameters caused confusion by creating a false association with HTTP controllers. However, interceptors are not tied to HTTP and are used universally. Now, instead of `$controller` and `$action`, we use the Target definition, which can point to more than just class methods.

The `$parameters` parameter, which is a list of arguments, has now been moved to the `CallContextInterface` invocation context.

```php
/** @deprecated Use InterceptorInterface instead */
interface CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed;
}

interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

### CallContextInterface

The `CallContextInterface` invocation context contains all the information about the call:
- Target — the definition of the call target.
- Arguments — the list of arguments for the call.
- Attributes — additional context that can be used to pass data between interceptors.

> [!NOTE]
> `CallContextInterface` is immutable.

### TargetInterface

`TargetInterface` defines the target whose call we want to intercept.

If you need to replace the Target in the interceptor chain, use the static methods of the `\Spiral\Interceptors\Context\Target` class to create a new Target.

The basic set includes several types of Target:
- `Target::fromReflectionMethod(ReflectionFunctionAbstract $reflection, class-string|object $classOrObject)` \
  Creates a Target from a method reflection. The second argument is mandatory because the method reflection may refer to a parent class.
- `Target::fromReflectionFunction(\ReflectionFunction $reflection, array $path = [])` \
  Creates a Target from a function reflection.
- `Target::fromClosure(\Closure $closure, array $path = [])` \
  Creates a Target from a closure. Use PHP 8 syntax for better clarity:
    ```php
    $target = Target::fromClosure($this->someAction(...));
    ```
- `Target::fromPathString(string $path, string $delimiter = '.')` and
  `Target::fromPathArray(array $path, string $delimiter = '.')` \
  Creates a Target from a path string or array. In the first case, the path is split by the delimiter; in the second, the delimiter is used when converting the Target to a string. This type of Target without an explicit handler is used for RPC endpoints or message queue dispatching.
- `Target::fromPair(string|object $controller, string $action)` \
  An alternative way to create a Target from a controller and method, as in the old interface. The method will automatically determine how the Target should be created.

### Compatibility

Spiral 3.x will work as expected with both old and new interceptors. However, new interceptors should be created based on the new interface.

In Spiral 4.x, support for old interceptors will be disabled. You will likely be able to restore it by including the `spiral/hmvc` package.

### Building an Interceptor Chain

If you need to manually build an interceptor chain, use `\Spiral\Interceptors\PipelineBuilderInterface`.

In Spiral v3, two implementations are provided:
- `\Spiral\Interceptors\PipelineBuilder` — an implementation for new interceptors only.
- `\Spiral\Core\CompatiblePipelineBuilder` — an implementation from the `spiral/hmvc` package that supports both old and new interceptors simultaneously.

> [!NOTE]
> In Spiral 3.14, the implementation for `PipelineBuilderInterface` is not defined in the container by default.
> `CompatiblePipelineBuilder` is used in Spiral v3 services as a fallback implementation.
> If you define your own implementation, it will be used instead of the fallback implementation in all framework pipelines.

At the end of the interceptor chain, there should always be a `\Spiral\Interceptors\HandlerInterface`, which will be called if the interceptor chain does not terminate with a result or exception.

The `spiral/interceptors` package provides several basic handlers:
- `\Spiral\Interceptors\Handler\CallableHandler` — simply calls the callable from the Target "as is".
- `\Spiral\Interceptors\Handler\AutowireHandler` — calls a method or function, resolving missing arguments using the container.

```php
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\CallableHandler;

$interceptors = [
    new MyInterceptor(),
    new MySecondInterceptor(),
    new MyThirdInterceptor(),
];

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(...$interceptors)
    ->build(handler: new CallableHandler());

$pipeline->handle(new CallContext(
    target: Target::fromPair($controller, $action),
    arguments: $arguments,
    attributes: $attributes,
));
```
