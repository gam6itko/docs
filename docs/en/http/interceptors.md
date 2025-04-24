# HTTP — Interceptors

Spiral provides interceptors for HTTP requests that allow you to intercept and modify requests and responses at various 
points in the request lifecycle.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

## Domain Core Builder

The framework provides a convenient Bootloader called `Spiral\Bootloader\DomainBootloader` and allows developers to
register interceptors and add a common functionality to the application, such as logging, error handling, and security
measures, in a single place, rather than having to add them to each controller.

The bootloader also provides an ability to configure the order in which the interceptors are executed, allowing
developers to control the flow of the application.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        HandleExceptionsInterceptor::class,
        JsonPayloadResponseInterceptor::class,
    ];
}
```

## Examples

### Cycle Entity Resolution

The [Cycle Bridge](https://github.com/spiral/cycle-bridge/) package
provides `Spiral\Cycle\Interceptor\CycleInterceptor`.
Use `CycleInterceptor` to automatically resolve entity injections based on parameter values:

To activate the interceptor:

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        // ...
        CycleInterceptor::class,
    ];
}
```

You can use any cycle entity injection in your `UserController` methods, the `<id>` parameter will be used as the
primary key. If an entity can't be found, the 404 exception will be thrown.

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

use App\Domain\Blog\Entity\User;
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users/<id>')]
    public function show(User $user)
    {
        dump($user);
    }
}
```

> **See more**
> Read more about Annotated routes in the [HTTP — Routing](../http/routing.md#routes-based-on-attributes) section.

You must use named parameters if more than one entity is expected:

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use App\Domain\Blog\Entity\Blog;
use App\Domain\Blog\Entity\Author;
use Spiral\Router\Annotation\Route;

final class BlogController
{
    #[Route(route: '/blog/<author>/<post>')]
    public function show(Author $author, Blog $post)
    {
        dump($author, $blog);
    }
}
```

> **Note**
> Method arguments must be named as route parameters.

### Guard Interceptor

Use `Spiral\Domain\GuardInterceptor` to implement RBAC pre-authorization logic (make sure to install and
activate `spiral/security`).

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        // ...
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', Rule\AllowRule::class);
        $rbac->associate(Guest::ROLE, 'home.about', Rule\ForbidRule::class);
    }
}
```

You can use attributes to configure what permissions to apply for the controller action:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    #[Guarded(permission: 'home.index')]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(permission: 'home.about')]
    public function about(): string
    {
        return 'OK';
    }
}
```

To specify a fallback action when the permission is not checked, use `else` attribute of `Guarded`:

```php app/src/Endpoint/Web/HomeController.php
#[Guarded(permission: 'home.about', else: 'notFound')]
public function about(): string
{
    return 'OK';
}
```

> **Note**
> Allowed values: `notFound` (404), `forbidden` (401), `error` (500), `badAction` (400).

Use the attribute `Spiral\Domain\Annotation\GuardNamespace` to specify controller RBAC namespace and remove a prefix
from every action. You can also skip the permission definition in `Guarded` when a namespace is specified (security
component will use `namespace.methodName` as a permission name).

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Domain\Annotation\Guarded;
use Spiral\Domain\Annotation\GuardNamespace;

#[GuardNamespace(namespace: 'home')]
class HomeController
{
    #[Guarded]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(else: 'notFound')]
    public function about(): string
    {
        return 'OK';
    }
}
```

#### Rule Context

You can use all method parameters as rule context, for example, we can create a rule:

```php app/src/Application/Security/SampleRule.php
namespace App\Application\Security;

use Spiral\Security\ActorInterface;
use Spiral\Security\RuleInterface;

class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['user']->getID() !== 1;
    }
}
```

To activate the rule:

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use App\Application\Security\SampleRule;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        //...
        CycleInterceptor::class,
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', SampleRule::class);
        $rbac->associate(Guest::ROLE, 'home.about', Rule\ForbidRule::class);
    }
}

```

> **Note**
> Make sure that the route includes `<id>` or `<user>` parameter.

And modify the method:

```php
#[Guarded] 
public function index(User $user): string
{
    return 'OK';
}
```

The method would not allow invoking the method with user id `1`.

> **Note**
> Make sure to enable `CycleInterceptor` before `GuardInterceptor` in domain core.

### DataGrid Interceptor

You can automatically apply datagrid specifications to an iterable output using `DataGrid` attribute
and `GridInterceptor`.
This interceptor is called after the endpoint invocation because it uses the output.

```php app/src/Endpoint/Web/UsersController.php
use App\Domain\User\Repository\UserRepository;
use App\Intergarion\Keeper\View\UserGrid;
use Spiral\DataGrid\Annotation\DataGrid;
use Spiral\Router\Annotation\Route;

class UsersController
{
    #[Route(route: '/users', name: 'users')]
    #[DataGrid(grid: UserGrid::class)]
    public function list(UserRepository $userRepository): iterable
    {
        return $userRepository->select();
    }
}   
```

> **Note**
> `grid` property should refer to a `GridSchema` class with specifications declared in the constructor.

```php app/src/Intergarion/ViewKeeper/Keeper/UserGrid.php
namespace App\Intergarion\Keeper\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter;
use Spiral\DataGrid\Specification\Value;

class UserGrid extends GridSchema
{
    public function __construct()
    {
        $this->addSorter('email', new Sorter\Sorter('email'));
        $this->addSorter('name', new Sorter\Sorter('name'));
        $this->addFilter('status', new Filter\Equals('status', new Value\EnumValue(new Value\StringValue(), 'active', 'disabled')));
        $this->setPaginator(new PagePaginator(20, [10, 20, 50, 100]));
    }
}
```

Optionally, you can specify `view` property to point to a callable presenter for every record.
Without specifying it `GridInterceptor` will call `__invoke` in the declared grid.

```php app/src/Intergarion/ViewKeeper/Keeper/UserGrid.php
namespace App\Application\View;

use Spiral\DataGrid\GridSchema;
use App\Database\User;

class UserGrid extends GridSchema
{
    //...
    
    public function __invoke(User $user): array
    {
        return [
            'id'     => $user->id,
            'name'   => $user->name,
            'email'  => $user->email,
            'status' => $user->status
        ];
    }
}
```

You can specify grid defaults (such as default sorting, filtering, pagination) via `defaults` property or
using `getDefaults()` method in your grid:

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(
    grid: UserGrid::class,
    defaults: [
        'sort' => ['name' => 'desc'],
        'filter' => ['status' => 'active'],
        'paginate' => ['limit' => 50, 'page' => 10]
    ]
)]
```

By default, grid output will look like this:

```json
{
  "status": 200,
  "data": [
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ]
}
```

You can rename `data` property or pass the exact `status` code `options` or `getOptions()` method in the grid:

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(grid: UserGrid::class, options: ['status' => 201, 'property' => 'users'])]
```

```json
{
  "status": 201,
  "users": [
    ...
  ]
}
```

`GridInterceptor` will create a `GridFactoryInterface` instance to wrap the given iterable source with the declared grid
schema. `GridFactory` is used by default, but if you need more complicated logic, such as using a custom counter or
specifications utilization, you can declare your own factory in the annotation:

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(grid: UserGrid::class, factory: InheritedFactory::class)]
```

### Pipeline Interceptor

This interceptor allows customising endpoint interceptors using the `Pipeline` attribute.
When declared in the domain core interceptors list, this interceptor injects specified annotated interceptors on the
position where the `PipelineInterceptor` is declared.

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        Domain\PipelineInterceptor::class, //all annotated interceptors go here
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

`Pipeline` attribute allows skipping subsequent interceptors:

```php
#[Pipeline(pipeline: [OtherInterceptor::class], skipNext: true)]
public function action(): string
{
    //
}
 ```

Using the prev bootloader, we will get the next interceptors list:

- Spiral\Cycle\Interceptor\CycleInterceptor
- OtherInterceptor

> **Note**
> All interceptors after `PipelineInterceptor` will be omitted.

### Use cases

For example, it can be helpful when an endpoint should not apply any interceptor or not all of them are currently
required:

```php
#[Route(route: '/show/<user:int>/email/<email:int>', name: 'emails')]
#[Pipeline(pipeline: [CycleInterceptor::class, GuardInterceptor::class], skipNext: true)]
public function email(User $user, Email $email, EmailFilter $filter): string
{
    $filter->setContext(compact('user', 'email'));
    if (!$filter->isValid()) {
        throw new ForbiddenException('Email doesn\'t belong to a user.');
    }
    //...
}
 ```

> **Note**
> `FilterInterceptor` should not be applied here because of a complicated context, so we set it manually and call a
> custom `isValid()` check. Also, `GridInterceptor` is redundant here.

To have full control over the interceptors list, you need to specify `PipelineInterceptor` as the first one.

### All Together

Use all interceptors together to implement rich domain logic and secure controller actions:

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

## Route Specific Interceptors

To activate a interceptors pipeline for a specific route, you can build a pipeline using the
`Spiral\Interceptors\PipelineBuilder` class and register it with the route using the `withHandler()` method.

```php
// Create a pipeline for a specific route
$pipeline = (new PipelineBuilder())
    ->withInterceptors(new CustomInterceptor())
    ->build(new CallableHandler());

// Register the route with the custom pipeline
$router->setRoute(
    'home',
    new Route(
        '/home/<action>',
        (new Controller(HomeController::class))->withHandler($pipeline)
    )
);
```

## Parameter Type Conversion

### Integer Values Conversion

If you want to use typed route parameters injection in controllers such as `function user(int $id)`, you need to create an interceptor to handle the type conversion:

```php
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

/**
 * Converts all numeric string arguments to integers.
 */
class ParameterTypeCastInterceptor implements InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $arguments = $context->getArguments();

        foreach ($arguments as $key => $value) {
            if (\is_string($value) && \ctype_digit($value)) {
                $arguments[$key] = (int) $value;
            }
        }

        return $handler->handle($context->withArguments($arguments));
    }
}
```

### Value Objects Conversion

For handling complex type conversions, such as converting string UUIDs to UUID objects:

```php
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class UuidParameterConverterInterceptor implements InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $arguments = $context->getArguments();
        $target = $context->getTarget();
        $reflection = $target->getReflection();

        if ($reflection === null) {
            // If reflection is not available, just call the handler
            return $handler->handle($context);
        }

        // Iterate all action arguments
        foreach ($reflection->getParameters() as $parameter) {
            $paramName = $parameter->getName();
            $paramType = $parameter->getType();

            // If parameter exists in arguments and has UuidInterface type hint
            if (isset($arguments[$paramName]) &&
                $paramType !== null &&
                $paramType->getName() === UuidInterface::class
            ) {
                try {
                    // Replace argument value with Uuid instance
                    $arguments[$paramName] = Uuid::fromString($arguments[$paramName]);
                } catch (\Throwable $e) {
                    // Handle invalid UUID format if needed
                }
            }
        }

        return $handler->handle($context->withArguments($arguments));
    }
}
```