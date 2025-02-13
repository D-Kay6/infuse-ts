[![License: MIT](https://img.shields.io/npm/l/infuse-ts.svg)](https://github.com/D-Kay6/Infuse-TS/blob/master/LICENSE)
[![NPM version](https://img.shields.io/npm/v/infuse-ts.svg)](https://www.npmjs.com/package/infuse-ts)
[![NPM](https://img.shields.io/npm/dm/infuse-ts.svg)](https://www.npmjs.com/package/infuse-ts)
[![Node version](https://img.shields.io/node/v/infuse-ts.svg)](https://img.shields.io/node/v/infuse-ts.svg)

# Infuse-TS
A pure TypeScript DI container that supports both decorators and manual registration without relying on experimental features.
Has support for context-based scoping and aliasing.

## Install
```sh
npm install infuse-ts
```

## Usage
There are two main ways you can use Infuse-TS: with or without decorators.

### With Decorators
You can use decorators to register dependencies in a more concise way.
The scope is optional and defaults to `Scope.Transient`.

#### Direct

```typescript
import { Injectable, Scope } from 'infuse-ts';

@Injectable()
export class MyService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

@Injectable(Scope.Request, MyService)
export class MyController {
  private readonly myService: MyService;

  constructor(myService: MyService) {
    this.myService = myService;
  }
}
```

You can also use the `@Inject` decorator to Inject dependencies into a class field.

```typescript
import { Injectable, Inject, Scope } from 'infuse-ts';

@Injectable()
export class MyService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

@Injectable(Scope.Transient)
export class MyController {
  @Inject(MyService)
  private readonly myService!: MyService;
}
```

#### Alias
You can register classes as a child of another class using the `@InjectableAs` decorator.
This allows you to resolve a collection of all classes that are children of a parent class.
You can define the dependency as a tuple with the parent class as the only element.

```typescript
import { Injectable, InjectableAs } from 'infuse-ts';

export class BaseService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

@InjectableAs(BaseService)
export class Service1 extends BaseService {
  public doSomething() {
    console.log('Goodbye, World!');
  }
}

@InjectableAs(BaseService)
export class Service2 extends BaseService {
  public doSomething() {
    console.log('Farewell, World!');
  }
}

@Injectable([BaseService])
export class MyController {
  private readonly services: BaseService[];

  constructor(services: BaseService[]) {
    this.services = services;
  }
}
```

### Without Decorators
In order to register a service without decorators, you must use the `Container` class directly.
The `Container` class has a static property called `default` that holds a singleton instance of the container, but you can also create your own instance of the container.

Each one of the registration methods returns a registration object to define how the dependency should be resolved.
You must always call either of the `as*` methods to actually register the dependency.

#### Registering Components
A component is an alias for a constructable object. You can register a component using the `registerComponent` method.
The `registerComponent` method takes a component and a list of the constructor arguments.

```typescript
import { Container } from 'infuse-ts';

export class MyService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

export class MyController {
  private readonly myService: MyService;
  
  constructor(myService: MyService) {
    this.myService = myService;
  }
}

Container.default.registerComponent(MyService).instancePerDependency().asSelf();
Container.default.registerComponent(MyController, MyService).instancePerRequest().asSelf();
```

#### Registering Instances
You can register an instance of a class using the `registerInstance` method.
The `registerInstance` method takes a class and an instance of that class.

```typescript
import { Container } from 'infuse-ts';

export class MyService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

const myService = new MyService();
Container.default.registerInstance(MyService, myService).asSelf();
```

#### Registering Factories
You can register a factory using the `registerFactory` method.
The `registerFactory` method takes a class and a factory function that returns an instance of that class.
The first argument of the factory function is the container for resolving dependencies.

```typescript
import { Container } from 'infuse-ts';

export class MyService {
  public doSomething() {
    console.log('Hello, World!');
  }
}

Container.default.registerFactory(MyService, (container) => new MyService()).instancePerDependency().asSelf();
```

#### Resolving
There are two ways to resolve a dependency: using the `resolve` method or using the `resolveRequired` method.
The `resolve` method returns `undefined` if the dependency is not registered, while the `resolveRequired` method throws an `NotRegisteredError` error if the dependency is not registered.

```typescript
import { Container } from 'infuse-ts';

export abstract class BaseService {
  public abstract doSomething();
}

export class Service1 extends BaseService {
  public doSomething() {
    console.log('I am Service1');
  }
}

export class Service2 extends BaseService {
  public doSomething() {
    console.log('I am Service2');
  }
}

export class Service3 extends BaseService {
  public doSomething() {
    console.log('I am Service3');
  }
}

Container.default.registerComponent(Service1).instancePerDependency().as(BaseService).asSelf();
Container.default.registerComponent(Service2).instancePerDependency().as(BaseService);

// Using resolve
const resolveOne = Container.default.resolve(Service1); // returns an instance of Service1
const resolveFirst = Container.default.resolve(BaseService); // returns an instance of the first registered service, Service1
const resolveMany = Container.default.resolve([BaseService]); // returns an array with instances of Service1 and Service2
const resolveMissing = Container.default.resolve(Service3); // returns undefined

// Using resolveRequired
const requireOne = Container.default.resolveRequired(Service1); // returns an instance of Service1
const requireFirst = Container.default.resolveRequired(BaseService); // returns an instance of the first registered service, Service1
const requireMany = Container.default.resolveRequired([BaseService]); // returns an array with instances of Service1 and Service2
const requireMissing = Container.default.resolveRequired(Service3); // throws an error
```

### Request Scope
The request scope depends on the [cls-hooked](https://www.npmjs.com/package/cls-hooked) package to provide context-based scoping.
Please refer to its documentation for any specifics.

To keep it simple, anything running within a request context will have access to the same instances of dependencies registered as `Scope.Request`.
```typescript
import { Container, Injectable, RequestContext, Scope } from 'infuse-ts';

@Injectable(Scope.Request)
export class MyService {
  private sum: number = 0;
  public add(value: number) {
    this.sum += value;
    console.log(this.sum);
  }
}

RequestContext.run(() => {
  const service1 = Container.default.resolve(MyService);
  service1.add(1); // prints: 1
  
  const service2 = Container.default.resolve(MyService);
  service2.add(2); // prints: 3
});

RequestContext.run(() => {
  const service3 = Container.default.resolve(MyService);
  service3.add(3); // prints: 3
});

```

### Caveats
There are a few caveats to be aware of when using Infuse-TS.
- Circular dependencies are not supported.
- Modules need to be imported for the decorators to work. This is a limitation of TypeScript.

## Background
I'm originally a C# developer, and I've been using DI containers for many years. When I started working with TypeScript halfway through 2024, I initially had to learn how to work with the type system and the language itself.
A couple of months later, I was starting to miss the ease of working with DI containers in C#. However, I found that most DI containers for TypeScript were using the experimental Decorators API, even though TypeScript added support for decorators in version 5.0.
I wanted to create a library that didn't rely on experimental features but still offered the ease of working with decorators.
That's how Infuse-TS was born. It uses the TS5.0 decorators to provide a similar API to other DI containers, but without the need for experimental features.
I know there is something to be said about not using DI containers in JavaScript/TypeScript, but I find them to be a useful tool when used correctly.
