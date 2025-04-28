### Client

The responsibility for making client gRPC calls has been moved to the `spiral/grpc-client` package, which is included by default.

Generating client classes is no longer required, as `spiral/grpc-client` generates the necessary proxy classes at runtime.
That's why all the extra generators (BootloaderGenerator, ConfigGenerator, ClientGenerator) and client-related classes have been removed from this package.

To configure `spiral/grpc-client`, a `client` section has been added to the gRPC configuration:

```php
return [
    // File 'app/config/grpc.php'
    // ... other options ...

    'client' => new GrpcClientConfig(
        interceptors: [
            SetTimoutInterceptor::createConfig(6_000),
            RetryInterceptor::createConfig(
                maximumAttempts: 1,
            ),
            ExecuteServiceInterceptors::class,
        ],
        services: [
            new \Spiral\Grpc\Client\Config\ServiceConfig(
                connections: ConnectionConfig::createInsecure('localhost:9001'),
                interfaces: [
                    \GRPC\Mailer\MailerServiceInterface::class,
                    \GRPC\Mailer\PingerServiceInterface::class,
                ],
            ),
        ],
    )
];
```
