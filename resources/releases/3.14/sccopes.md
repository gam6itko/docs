## Container Scopes

Container Scopes are integrated even deeper.
In Spiral, each type of worker is handled by a [separate dispatcher](https://spiral.dev/docs/framework-dispatcher).
Each dispatcher has its own scope, which can be used to limit the set of services available to the worker.

During request processing, such as HTTP, the context (`ServerRequestInterface`) passes through a middleware pipeline.
At the very end, when the middleware has finished processing and the controller has not yet been called, there is a moment when the request context is finally prepared.
At this moment, the contextual container scope (in our case, `http-request`) is opened, and the `ServerRequestInterface` is placed in the container.
In this scope, interceptors come into play, after which the controller is executed.

As before, you can additionally open scopes in middleware, interceptors, or the business layer, for example, to limit the authorization context in a multi-tenant application.

You can view the names of dispatcher scopes and their contexts in the enum `\Spiral\Framework\Spiral`.