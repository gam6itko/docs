# GRPC — Interceptors

Spiral provides interceptors for gRPC services that allow you to intercept and modify requests and responses at various 
points in the request lifecycle.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

There are two types of interceptors:

1. Server interceptors
2. Client interceptors

## Server Interceptors

Server interceptors are used to intercept and modify requests and responses received by a server. They are typically
used to add cross-cutting functionality such as logging, authentication, or monitoring to the server.

### Logging Interceptor

Here is an example of a simple interceptor that logs a request before and after processing:

```php
namespace App\Endpoint\GRPC\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class LogInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();

        $this->logger->info('Request received...', [
            'target' => (string) $target,
            'path' => $target->getPath(),
        ]);

        $response = $handler->handle($context);

        $this->logger->info('Request processed', [
            'target' => (string)$target,
            'path' => $target->getPath(),
        ]);

        return $response;
    }
}
```

### Exception Handler Interceptor

Here is an example of a simple interceptor that handles exceptions thrown by the server. It will catch all exceptions
and convert them to a gRPC exception.

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;
use Spiral\RoadRunner\GRPC\Exception\GRPCExceptionInterface;

final class ExceptionHandlerInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        try {
            return $handler->handle($context);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            if ($e instanceof GRPCExceptionInterface) {
                throw $e;
            }

            throw new GRPCException(
                message: $e->getMessage(),
                previous: $e,
            );
        }
    }
}
```

### Receiving trace context from request

Here is an example of a simple interceptor that receives trace context from the request.

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\TracerFactoryInterface;

class InjectTelemetryFromContextInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly TracerFactoryInterface $tracerFactory,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $ctx = $context->getArguments()[0] ?? null;

        $traceContext = $ctx instanceof \Spiral\RoadRunner\GRPC\ContextInterface
            ? (array) $ctx->getValue('telemetry-trace-id')
            : [];

        $target = $context->getTarget();

        return $this->tracerFactory->make($traceContext)->trace(
            name: \sprintf('Interceptor [%s]', __CLASS__),
            callback: static fn(): mixed => $handler->handle($context),
            attributes: [
                'target' => (string) $target,
                'path' => $target->getPath(),
            ],
            scoped: true,
            traceKind: TraceKind::SERVER,
        );
    }
}
```

### Guard Interceptor

Here is an example of a simple interceptor that checks if the user is authenticated. It will use PHP attributes to
determine which methods require authentication. An authentication token is passed in the request metadata.

```php
namespace App\Endpoint\GRPC\Interceptor;

use App\Attribute\Guarded;
use Spiral\Attributes\ReaderInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

final class GuardedInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $reflection = $context->getTarget()->getReflection();
        
        if ($reflection !== null) {
            $attribute = $this->reader->firstFunctionMetadata($reflection, Guarded::class);

        if ($attribute !== null) {
                $ctx = $context->getArguments()[0] ?? null;
                if ($ctx instanceof ContextInterface) {
                    $this->checkAuth($attribute, $ctx);
                }
            }
        }

        return $handler->handle($context);
    }

    private function checkAuth(Guarded $attribute, ContextInterface $ctx): void
    {
        // Metadata always stores values as array. 
        $token = $ctx->getValue($attribute->tokenField)[0] ?? null;

        // Here you can implement your own authentication logic
        if ($token !== 'secret') {
            throw new \Exception('Unauthorized.');
        }
    }
}
```

And example of a method that requires authentication:

```php
use App\Attribute\Guarded;

#[Guarded]
public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    // ...
}
````

And example of Guarded attribute:

```php
namespace App\Attribute;

use Doctrine\Common\Annotations\Annotation\NamedArgumentConstructor;

#[\Attribute(\Attribute::TARGET_METHOD), NamedArgumentConstructor]
class Guarded
{
    public function __construct(
        public readonly string $tokenField = 'token'
    ) {
    }
}
```

### Registering Interceptors

To use this interceptor, you will need to register them in the configuration file `app/config/grpc.php`.

```php app/config/grpc.php
return [    
    'interceptors' => [
        \App\Endpoint\GRPC\Interceptor\LogInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\ExceptionHandlerInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\GuardedInterceptor::class,
    ]
];
```

## Client Interceptors

Client interceptors are used to intercept and modify requests and responses sent by a client. They are typically used to
add cross-cutting functionality such as logging, modifying header, handling response errors.

### Interceptable Client class

If you want to use client interceptors, you will need to modify a client class from [client SDK](./client.md) section.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Service\PingerClient;
use App\Service\RequestCore;
use GRPC\Pinger\PingerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Core\InterceptableCore;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        PingerInterface::class => [self::class, 'initPingService'],
    ];

    private function initPingService(
        EnvironmentInterface $env
    ): PingServiceInterface
    {
        $core = new InterceptableCore(
            new RequestCore(
                $env->get('PING_SERVICE_HOST', '127.0.0.1:9001'),
                ['credentials' => \Grpc\ChannelCredentials::createInsecure()]
            )
        );

        // Here you can register your interceptors
        $core->addInterceptor(new \App\Service\Interceptor\HandleResponseErrorsInterceptor());

        return new PingerClient($core);
    }
}
```

And then implement the `RequestCore` class:

```php
namespace App\Service;

use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

final class RequestCore extends \Grpc\BaseStub implements CoreInterface
{
    public function callAction(string $controller, string $action, array $parameters = []): mixed
    {
        $ctx = $parameters['ctx'];
        \assert($ctx instanceof ContextInterface);

        return $this->_simpleRequest(
            $action,
            $parameters['in'],
            [$parameters['responseClass'], 'decode'],
            (array) $ctx->getValue('metadata'),
            (array) $ctx->getValue('options')
        )->wait();
    }
}
```

And finally modify the `PingerClient` class:

```php
namespace App\Service;

use App\GRPC\Pinger;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class PingerClient implements Pinger\PingerInterface
{
    public function __construct(
        private readonly RequestCore $core
    ) {
    }

    public function ping(GRPC\ContextInterface $ctx, Pinger\PingRequest $in): Pinger\PingResponse
    {
        return $this->sendRequest(
            '/' . self::NAME . '/ping',
            $in,
            $ctx,
            Pinger\PingResponse::class
        );
    }

    /**
     * @template T of object
     * @param non-empty-string $method
     * @param class-string<T> $response
     * @return T
     */
    public function sendRequest(
        string $method,
        \GRPC\Ping\PingRequest $in,
        GRPC\ContextInterface $ctx,
        string $response
    ): object {
        [$response, $status] = $this->core->callAction(
            self::class, $method,
            [
                'responseClass' => $response,
                'ctx' => $ctx,
                'in' => $in,
            ]
        );

        return $response;
    }
}
```

### Handle Response Errors Interceptor

```php
namespace App\Service\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class HandleResponseErrorsInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        [$response, $status] = $core->callAction($controller, $action, $parameters);

        $code = $status->code ?? GRPC\StatusCode::UNKNOWN;

        if ($code !== GRPC\StatusCode::OK) {
            throw new GRPC\Exception\GRPCException(
                message: $status->details,
                code: $status->code
            );
        }

        return [$response, $code];
    }
}
```

### Passing telemetry trace ID to the context

```php
namespace App\Service\Interceptor;

use Psr\Container\ContainerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;
use Spiral\RoadRunner\GRPC\ResponseHeaders;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\TracerInterface;
use Spiral\Core\CoreInterface;

class InjectTelemetryIntoContextInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $tracer = $this->container->get(TracerInterface::class);
        \assert($tracer instanceof TracerInterface);

        if (isset($parameters['ctx']) and $parameters['ctx'] instanceof RequestContext) {
            $metadata = $parameters['ctx']->getValue('metadata');
            if(!\is_array($metadata)) {
                $metadata = [];
        }

            $metadata['telemetry-trace-id'] = $tracer->getContext();
            $parameters['ctx'] = $parameters['ctx']->withValue('metadata', $metadata);
        }

        return $tracer->trace(
            name: \sprintf('GRPC request %s', $action),
            callback: static fn() => $core->callAction($controller, $action, $parameters),
            attributes: compact('controller', 'action'),
            traceKind: TraceKind::PRODUCER
        );
    }
}
```