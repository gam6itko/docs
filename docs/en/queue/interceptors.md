# Queue — Interceptors

Spiral provides a way for developers to customize the behavior of their job processing pipeline through the
use of interceptors. An interceptor is a piece of code that is executed before or after a job is pushed or consumed, and
which allows developers to hook into the job processing pipeline to perform some action.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

Interceptors can be useful for a variety of purposes, such as handling errors, adding additional context to the job, or
performing some other action based on the job being processed.

To use interceptors you will need to implement the `Spiral\Interceptors\InterceptorInterface` interface. This interface
requires you to implement the `intercept` method.

## Interceptor for Push

Here is an example of a simple interceptor that logs a message before and after a job is pushed:

```php
namespace App\Application\Job\Interceptor;

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

        $this->logger->info('Job pushing...', [
            'target' => (string) $target,
        ]);

        $id = $handler->handle($context);

        $this->logger->info('Job pushed', [
            'id' => $id,
            'target' => (string) $target,
        ]);

        return $id;
    }
}
```

To use this interceptor, you will need to register it in the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\Application\Job\Interceptor\LogInterceptor;

return [    
    'interceptors' => [
        'push' => [
            LogInterceptor::class,
        ],
        'consume' => [
            //...
        ],
    ],
    // ...
];
```

> **Note**
> The handler's `handle` method will push and return the `id` string.

## Interceptor for Consume

To create an interceptor, you need to create a class and implement the interface `Spiral\Interceptors\InterceptorInterface`.
The handler's `handle` method will execute a consuming job, it doesn't return anything.

Let's create an `JobExceptionsHandlerInterceptor` that reports exceptions when a job fails:

```php
namespace App\Application\Job\Interceptor;

use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class JobExceptionsHandlerInterceptor implements InterceptorInterface
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
            throw $e;
        }
    }
}
```

To use this interceptor, you will need to register it in the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\Application\Job\Interceptor\JobExceptionsHandlerInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    'interceptors' => [
        'consume' => [
            new Autowire(JobExceptionsHandlerInterceptor::class),
        ],
        //...
    ],
    // ...
];
```

## Registering a new Interceptor

There are several ways to add a new interceptor.

### Via configuration

Add it to the configuration file `app/config/queue.php`.

```php app/config/queue.php
use App\CustomInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  List of all interceptors
     * -------------------------------------------------------------------------
     */
     'interceptors' => [
        // interceptors for push
        'push' => [
            // via fully qualified class name
            CustomInterceptor::class,
        
            // via Autowire
            new Autowire(CustomInterceptor::class),     
            
            // via interceptor instance
            new CustomInterceptor(),
        ],
        
        // interceptors for consume
        'consume' => [
            // via fully qualified class name
            CustomInterceptor::class,
        
            // via Autowire
            new Autowire(CustomInterceptor::class),     
            
            // via interceptor instance
            new CustomInterceptor(),
        ],
     ],
];
```

#### Via QueueBootloader

Call the method `addPushInterceptor` to add an interceptor for the `push` action. Or call the
method `addConsumeInterceptor` to add an interceptor for the `consume` action in
the `Spiral\Queue\Bootloader\QueueBootloader` class.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\Bootloader\QueueBootloader;

class AppBootloader extends Bootloader
{
    public function boot(QueueBootloader $queue): void
    {
        // via fully qualified class name
        $queue->addPushInterceptor(CustomInterceptor::class);
        $queue->addConsumeInterceptor(CustomInterceptor::class);

        // via Autowire
        $queue->addPushInterceptor(new Autowire(CustomInterceptor::class));
        $queue->addConsumeInterceptor(new Autowire(CustomInterceptor::class));
        
        // via interceptor instance
        $queue->addPushInterceptor(new CustomInterceptor());
        $queue->addConsumeInterceptor(new CustomInterceptor());
    }
}
```

## Examples

### Retry policy for failed jobs

It provides a robust and flexible retry mechanism for failed jobs. Implementing a retry policy is essential in a
distributed system, as it ensures that transient issues, such as network failures or temporary resource unavailability,
do not cause the system to lose important messages or data.

The interceptor's primary responsibility is to execute the given job and monitor it for any exceptions. If an exception
is thrown, the interceptor logs the error, updates the job's attempt count, and throws a `RetryException` with the
updated attempt count and delay. This allows the job to be re-enqueued with the new information, ensuring that it is
retried as per the specified policy.

When the consumer encounters a RetryException, it understands that the job should be returned to the queue with the
updated options provided within the exception. This enables the job to be retried according to the specified policy,
improving the overall reliability of the system by allowing jobs to recover from transient issues.

```php
namespace App\Endpoint\Job\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;
use Spiral\Queue\Exception\RetryException;
use Spiral\Queue\Options;

final class RetryPolicyInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly ExceptionReporterInterface $reporter,
        private readonly int $maxAttempts = 3,
        private readonly int $delay = 5,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        try {
            return $handler->handle($context);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            $headers = $context->getAttribute('headers', []);
            $attempts = (int) ($headers['attempts'] ?? $this->maxAttempts);

            if ($attempts === 0) {
                $this->logger->warning('Attempt to execute job failed', [
                    'target' => (string) $context->getTarget(),
                ]);
                return null;
            }

            throw new RetryException(
                reason: $e->getMessage(),
                options: (new Options())->withDelay($this->delay)->withHeader('attempts', (string)($attempts - 1)),
            );
        }
    }
}
```

The interceptor will attempt to process the job a specified number of times (`$maxAttempts`) before giving up. It also
includes a configurable delay (`$delay`) between each retry attempt, which can be helpful in avoiding retry storms in
the system.

One of the key advantages of using this interceptor, is that it enables developers to handle exceptions and apply retry
policies in a centralized, consistent manner. This approach leads to cleaner, more maintainable code, as it abstracts
away the error-handling logic from individual job handlers.
