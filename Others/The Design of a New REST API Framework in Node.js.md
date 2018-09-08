# The Design of a New REST API Framework in Node.js

## Abstract

I am working on [Sandra](https://github.com/SANDRAProject) as a backend engineer, constructing a full-restified API. I found Koa is quite a satisfying solution, but not enough. As known to all, REST specification utilizes a set of HTTP features to accomplish schematic features. A good case in point is that if the client requests with `Accept: application/json`, then the server is expected to return a JSON content. However, in Koa, such features should be implemented on my own, and no other frameworks, even the Hapi, provides a simple enough API to fulfill the requirement. So I have decided to develop my own REST API framework, _Ulla_, both resolving such problem and utilizing the decorator features in TypeScript.

<!--more-->

## Routing

### Defining

Previously, we call functions on a _Router_ object to construct routes, but such method creates extra codes, so I think use _Decorators_ may be a better approach.

Take Koa for example:

```typescript
const router = new Router();
router.use(async (ctx, next) => {
  // some code
});
router.get("/", async ctx => {
  // some code
});
```

By contrast, my design is:

```typescript
@RouteSet("/users")
class UsersApi {
  @RouteUse("/")
  static middleware = async () => {
    // some code
  };
  @RouteGet("/")
  static index = async () => {
    // some code
  };
}
```

First, the developer creates a class representing a set of routes with the decorator `RouteSet`. The _RouteSet_ can be prefixed with a path, e.g. `@RouterSet("/users")`.

Then, the developer adds static functions to the _RouteSet_ for specific routes with the decorator `RouteGet`, `RoutePost`, `RoutePut`, `RouteDelete`, and `RouteCustomMethod`.

This design defines routes without any code or extra object creation, which clearifies the code and improves readability.

As JavaScript is a dynamic language, and the script cannot automatically discover and load all scripts, the developer is required to use `import` statements to explicitly load the routing scripts.

For example, add this line in the main entrypoint before importing _Ulla_:

```typescript
import "routes";
```

And `/routes/index.js` can be:

```typescript
import "./index";
import "./users";
```

### Handler

#### Accessing Request Data

Commonly, we can see a _context_ object or _request_ and _response_ object for a typical JavaScript server framework. Such design is quite vivid, but IMO, it increases code complexity, making the code less easy to understand.

I purpose to use _parameter decorators_ for parameters, and the route handler can just simply define any parameters it likes. For example:

```typescript
@RouteGet("/")
const index = async (@reqParam("id") id: number, @reqBody("filter") filter: IFilter) => {
  // some code
}
```

And when the function takes no parameters, the wrapper can be omitted.

```typescript
const index = async () => {
  // some code
};
```

When no wrapper defined, but parameters taken, the parameters will all be `undefined`.

This feature is especially useful when connecting DAO with REST APIs. For example:

```typescript
@RouteSet("/users")
class User {
  @RouteGet("/")
  static findAll = async () => {
    // some code
  };
}
```

This makes a connection between DAO and API, with no extra code.

#### Constructing Response

Most times, we may get confused when trying to modify a response which has been already sent, so by my design, there is only one chance to respond data. The framework will take the return value of the handler function and intercept its content as the respond data.

The following code descibes what the return value should looks like.

```typescript
interface IReturn {
  code: number;
  headers: Map<string, string>;
  content: any;
}
```

For example, a typical 404 Not Found can be constructed with the following code:

```typescript
@RouteGet("/404")
const notFound = () => {
  code: 404,
  headers: {
	"x-handled-with": "404-route"
  },
  content: {
    code: "NOT_FOUND"
  }
}
```

## Middleware

### Fulfilling REST Specification

As described previously, handling HTTP schematic headers is part of the REST specification, but unfortunately, hardly does modern REST API framework implements these features, and the developers have to figure it out on their own. My design provides helpers to accomplish it, making fulfilling specification much easier, while keeping the framework still tiny and extensible.

#### Content Types

Take handling various content MIME types for example, the developer may load a external module to handle the serialization and deserialization.

```typescript
import { jsonSerializer, jsonDeserializer } from "ulla-content-json";

const server = new UllaApp();
server.loadContentDeserializer("application/json", jsonDeserializer);
server.loadContentSerializer("application/json", jsonSerializer);
server.setDefaultResponseType("application/json");
```

By such approach, the REST API server is able to handle JSON request contents and return JSON responses.

As for a specific route, the handler is expected to return a valid JavaScript object, and the object will be automatically serialized with the corresponding serializer as the request header `Accept`.

#### Authorization

HTTP specification defines various authorization types, for example, Basic Auth and Bearer Auth. These headers should be also parsed.

```typescript
import basicAuthParser from "ulla-auth-basic";
import bearerAuthParser from "ulla-auth-bearer";
import User from "./models/user";

const server = new UllaApp();
server.loadAuthoizationHandler("basic", basicAuthParser, async auth => {
  // sample code
  const user = await User.findOne({
    username: auth.username,
    password: auth.password,
  });
  if (!user) throw new Error("username or password mismatch");
  else return { user };
});
server.loadAuthoizationHandler("bearer", bearerAuthParser, async auth => {
  // some code
});
```

Developers may define chained handler for an authorization type, and the returning value is passed as the first parameter of each handler. The return value of the last handler is sent to `Request.state`.

#### i18n

HTTP has also a header of `Accpet-Language`, which is quite useful for i18n. Tranditionally, developers have to write their own middleware for i18n, but I think these works can be done more simpler.

```typescript
// /lib/i18n
import LocalizedString from "ulla-i18n-string";
const localizedString = new LocalizedString();
LocalizedString.set("hello world!", {
  en_US: "hello world!",
  zh_CN: "你好世界！",
});
export default LocalizedString;

// /routes/index
import LocalizedString from "../lib/i18n";

const index = async (id: number, filter: IFilter) =>
  new LocalizedString("hello world!");
```

The framework, will automatically call `LocalizedString#toString` with locale code as the first parameter.

As for dates, as there already exists `Date` in JavaScript, the framework will utilize it instead of creating another `LocalizedDate`.

#### Encoding

For `Accept-Encoding`, I think it is better practice to implement it on a load balancer, but in some cases, the support is also necessary in the framework. As content types, this is done via a similar way.

```typescript
import { gzipEncoder, gzipDecoder } from "ulla-encoding-gzip";

const server = new UllaApp();
server.loadEncoder("gzip", gzipEncoder);
server.loadDecoder("gzip", gzipDecoder);
```

### Global Middleware

Global middlewares are of great usage, so the framework must provide such features. As middlewares differs from route handlers, it is also defined with a different way. IMO, such defining method is more readable.

```typescript
const server = new UllaApp();
server.use(async (req, next) => {
  // some code
});
```

### Route/RouteSet Middleware

Sometimes, we also want middlewares applied at Route or _RouteSet_ level, but as a different approach is used to define routes, the definition of middleware at Route or _RouteSet_ level cannot be the same as global middlewares.

I have figured out a seem-complex approach, to define middlewares in decorators. The developer may apply middlewares as below:

```typescript
const // some code
index = async () => {
  // some code
};
```

Moreover, according to the DRY priciple, I think middlewares can be pre-defined.

```typescript
const auth = UllaApp.buildMiddleware(
  (authType: string) => async (req, next) => {
    // some code
  },
);
```

Take note that `UllaApp.buildMiddleware` takes only one parameter of a function, which takes the parameter of the desired decorator and returns a middleware.

Then it can be applied to either a specific route or a _RouteSet_.

```typescript
@auth
@RouteSet("/")
class Index {
  // some code
}
```

# Ending

This is quite a simple design I want to accomplish, and as development gone through, I may add detailed information on the framework. If any suggestions, you are welcomed to contact me via email at the [about](/about/) page. Please look forward to my framework!
