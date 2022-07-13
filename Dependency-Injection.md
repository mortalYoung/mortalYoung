# Dependency Injection


## Abstract Layouts
For now, we have an web application which is mainly comprised of slots. 

For example, we could divide each page layout into several parts. Not only the IDE-like pages, but also some background management systems like library systems or something.

Let's make a specific example. Now we have an application with a header, a footer, a side menu, and the content. So now we can abstract each part into a service.

```js
class ApplicationService {};
class HeaderService {};
class SideMenuService {};
class FooterService {};
class ContentService {};
```

Now, we get these five services.

## Dependencies

As we known, these five services couldn't be independent. In our daily life, these five services are more like to be interdependence.

For example, the side menu's rendering always depends on the header. And the footer also depends on the header. The content depends on header and side menu.

So we get the dependencies between these five services.

```js
ApplicationService -> HeaderService, SideMenuService, FooterService, ContentService
HeaderService -> null
SideMenuService -> HeaderService
FooterService -> HeaderService
ContentService -> HeaderService, SideMenuService, FooterService
```

## Construct

There is a easy way to handle these dependencies —— directly initialize service with its dependency. But here I want to share another way to handle these dependencies —— auto injection.

I achieve it based on typeScript's decorator.

Now, I want the dependency aumatically injected into service. I just need to define the dependency with decorator.
```ts
class HeaderService {
    constructor(){
        console.log('init header');
    }
};

class SideMenuService {
    constructor(@IHeader header){
        console.log('init sidebar');
        console.log('header in sidemenu:', header);
    }
};

class FooterService {
    constructor(@IHeader header){
        console.log('init footer');
        console.log('header in footer:', header);
    }
};

class ContentService {
    constructor(@IHeader header, @ISideMenu sideMenu, @IFooter footer){
        console.log('init footer');
        console.log('header in content:', header);
        console.log('siderMenu in content:', sideMenu);
        console.log('footer in content:', footer);
    }
}

class ApplicationService {
    constructor(services){
        // store all services into this application
    }
}
```


## Achieve

First, achieve the decorators

```ts
const DI_DEPENDENCIES = 'abc$dep';

type IServiceIdentity = any;

namespace instantiation {
    // ensure the decorators globally unique
    const _serviceIds = new Map();

    export function createDecorator(serviceId: string): IServiceIdentity {
        if (_serviceIds.has(serviceId)) {
            return _serviceIds.get(serviceId)!;
        }

        const id = function (target: Record<string, any>, propertyKey: string | symbol, parameterIndex: number) {
            if (arguments.length !== 3) {
                throw new Error('@IServiceName-decorator can only be used to decorate a parameter');
            }
            target[DI_DEPENDENCIES] = target[DI_DEPENDENCIES] || [];
            target[DI_DEPENDENCIES].push({ serviceId, parameterIndex });
        }

        id.toString = () => serviceId;

        _serviceIds.set(serviceId, id);
        return id;
    }
}
```

When call `instantiation.createDecorator` method, I get a decorator. Except that, I still need a registry to ensure the services globally unique.

```ts
namespace standalone {
    const _registry = new Map();

    export function registerSingleton(id: IServiceIdentity, ctor: any) {
        _registry.set(id, ctor)
    }
}
```

And now we can put every services into registry. Create decorators at meanwhile.
```ts
const IHeader = instantiation.createDecorator('HeaderService');
standalone.registerSingleton(IHeader, HeaderService);

const ISideMenu = instantiation.createDecorator('SideMenuService');
standalone.registerSingleton(ISideMenu, SideMenuService);

const IFooter = instantiation.createDecorator('FooterService');
standalone.registerSingleton(IFooter, FooterService);

const IContent = instantiation.createDecorator('ContentService');
standalone.registerSingleton(IContent, ContentService);
```


## Create Application

I have to export a create method to initialize the application.

```ts

namespace standalone {
    const _registry = new Map();

    export function registerSingleton(id: IServiceIdentity, ctor: any) {
        _registry.set(id, ctor)
    }

    class InstantiationService {
        constructor(collections: Map<IServiceIdentity, any>) {
            // we should first initialize the services without dependency
            // and initialize the services with one dependency
            // and initialize the services with two dependency
            // and so on up to five, to six, to the max count of depedencies.
            const stack = [...collections];
            let mindep = 0;
            while (stack.length) {
                // get current collection's minimum depedencies count in stack
                mindep = Math.min(...stack.map(([_, ctor]) => ctor[DI_DEPENDENCIES]?.length || 0));

                const [id, ctor] = stack.pop();
                const deps: any[] = ctor[DI_DEPENDENCIES] || [];
                if (deps.length === mindep) {
                    deps.sort((a, b) => a.parameterIndex - b.parameterIndex);

                    const params = deps.map(({ serviceId }) => {
                        return this[serviceId];
                    })

                    this[id] = new ctor(...params);

                } else {
                    stack.unshift([id, ctor])
                }
            }
        }
    }

    export function create() {
        return new InstantiationService(_registry);
    }
}
```


## Last

Just to call `standalone.create();`.


[TS Playground](https://www.typescriptlang.org/play?#code/MYewdgzgLgBAIgSQPpwKIAVUDk1YMIKoDKMAvDAOQCGARsACQAmApgA4UDcAUF1AJ6tmMBEWYAnAG4BLYMwQswUKfzIwqYPty5gqAW2YRWVWTCmQo6pVSXgYAby4wnMAPQuYzSAFcxQqAAshFlAxaxAxCBgAcwAbEBoqGJi+GC8wKQBHL2ZHZ1BzGCQIcWlZeUjyMGYAdxgAWSpWAAoASi1nDwAPVnDYADM04BswGGBfa2Y4ZhCwsSbiyRk5RgAuGGgxMyiWtZESpflPJRUHDo6pPpgmov2yxggAOn8qCHnb5ZaW+1yzs98oHwjG6LO6PKLMKBvEEfACE3F+TgAvjwEaNwNBTIxVAMwEMpLYmhYxOCoGsAErTcKMAA8Gy2ABo1BoAHyM1hiECCMT8ADSzD4azpYCiMAAPus+LoaCAYmyqKF9FBxAgwCxOmswF4peIvqdUc4Lld5VEtUdHjFPFEAjAYaRyABmXU-fUdAIc2pVWqoMQcuYUAACe2hWD0zAAtMFwrNRuoYOBkjAaEIvMUsVAQDBI6ElWoYEYFRDxBQ2s7UciXU4iSSANqIFAYbC4AjEAC6qirENryDQmBw2GbRDbovF1Zb8IrHagXfrvabhEHD1YKf8TTs63e8jlBaVYhVapgiJLZZRCKkjAe6aIUE2wtUrTIzPX0Pk7QRwNKy0exUhCw-m8xR4Iv8gKYuOSJcMi2ihoYxhCNA6iMIk4BCHqTj5BiSC+FEUgbCklQ1PUjStK+TjMN0vQwDieK2FhOE7kQWwWumYBNGeuyiM+CjHHwjJDOEazqHwTqophzDYbhDzfqxjC8emYgtM6kEdMAMQvJEKrwYoUjWPiYAcR+3youh15eHxcygEk0zDBAawNKw1JBn+XHKDxTJ8MywkVsZ6wWMAADWqjVg8wUWRa1GQGOpYIkxMC6GYLCsKoAAMYH6tU-hSBaVzwf5DwWsKAROlFLpxaqbCqA0AQPKVTTBZJvl+dVRFNNWSCyeELZfKQj5mdOPaNv284tgA-HllrWsOMBJZ8JEVmh6KwNWZ7tWIbbkDljU9M0gFzfNBQJTZbmjqovV1v1fb4ENYojpFu0GpcTQHWNBX+GQdqxfFbCeXdHRPRAvRNE0VCMjQXWPlQi7yqGO57mRMBhomkPbsqZWdCWxW7d5+Z6BUmZsI8ujNWuv4HFih4PoZP2osBYgjAEOHViTdy3VTziHierMwPTEBLYwa0wJ6oxybVwXY7oEDo1TiIeDExSU5zG0PGkEAZX0kK8ytnUY8ec3lgievgc6ZE9NylGDMMozjEq96oX8EIgYLGkWFpOngPpSzXLRuE7QeEE8FwKlqTAAASzBUCwYjuyYtvGWIpnC99ZzoTKzB5SAURNBQZjKDAgTh0WPvIoiWjeQgof52IqhmJpVjDA8Yxh0qUwzMLFDlxHUfMMW3CaYhcRVA8Xv0YxELgE0ZdhxHjLt+InfowHqkQJEDEsHUnheJ38touYcdmU0gYz5XecRwJGiJ8p6Ip2nGdZ+ksAQGezAJGI3cY8nFrX5nx-iKYIwPyw+hNQrAoIyb+8lUpFxLgtYQK9mBr01FXcwlhtJ1wbhMZuUZW6wPgRvd4r9e5IQHkPWeI9mLj2wevRkFDNRzy0AvIOAAxEAIB6LvC3rHeO4R94TwrrnSe4hT5CS3hfSAV84g32zv0ZhO5X5GUvh-cRX9+GVzMJRaRAiQF8IroXCCUCCgICYSwn+5Bq7O1rrpeuVtJiUmzFwighjWHQnwc7PuyFB5iToiQ4UTEx4GPUWIRkDjZ7vHnoHJeMA8DgCVIoTeMcFq72FgfZRWiT5uUZIGahXh1iPxwYI9JfijGVz6P4wR588jyNToo2+OdimFNkQid+lT05KN4ao-I0TSSaLAT7ER-0FHNIoP-cQODf7bw6cAxkQycE9PKaI-pN9ak7lGe0o4Ey1GFJ0ZBUukTFBHEQTXFBFi0FNxsbMTOOyOmd2cQhQhqdiGR1Ib4i5RxGTPJiSEuhBD+6p2OcwYiQA)