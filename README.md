# GraphQL Schema Bindings


[![npm](https://img.shields.io/npm/v/graphql-schema-bindings.svg)](https://www.npmjs.com/package/graphql-schema-bindings)
[![Build Status](https://travis-ci.org/IBM/graphql-schema-bindings.svg?branch=master)](https://travis-ci.org/IBM/graphql-schema-bindings)
[![Coverage Status](https://coveralls.io/repos/github/IBM/graphql-schema-bindings/badge.svg?branch=master)](https://coveralls.io/github/IBM/graphql-schema-bindings?branch=master)

GraphQL Schema Bindings is a flexible and scalable GraphQL schema generator that exposes common object oriented patterns to your GraphQL server.

GraphQL Schema Bindings uses decorators to define the necessary metadata to map your native types to GraphQL types.

Use `@type` to mark a class to be exported to GraphQL.

```javascript
@type
class Resource {
    ...
}
```

Exported types need to have fields that GraphQL can access.

```javascript
@type
class Resource {
  @field(ID) id;
  @field(String) name;
}
```

Fields are automatically inherited from parent types.

```javascript
@type
class BaseResource {
  // context can be bound to a field
  @context context;

  @field(ID)
  get id() {
    return this.data.id;
  }

  constructor(data) {
    this.data = data;
  }
}

@type
class Resource extends BaseResource {
  /**
   * The field return type can be wrapped in a
   * thunk to handle circular dependencies.
   */
  @field(() => Resource)
  parent() {
    return this.context.getParent(this.id);
  }

  /**
   * The return type can be an array of items.
   */
  @field([Resource])
  children() {
    return this.context.getChildren(this.id);
  }
}
```

## XKCD Example

With this schema you are able to retrieve details for [XKCD](https://xkcd.com) comics. You can get the latest comic, a random comic, or a comic by ID (number).

```javascript
import { ApolloServer } from "apollo-server";
import axios from "axios";
import {
  type,
  field,
  arg,
  ID,
  Int,
  createSchema
} from "graphql-schema-bindings";

@type
class Comic {
  @field(ID)
  get id() {
    return this.number;
  }

  @field(Int)
  get number() {
    return this.data.num;
  }

  @field(String)
  get title() {
    return this.data.title;
  }

  @field(String)
  get image() {
    return this.data.img;
  }

  @field(String)
  get transcript() {
    return this.data.transcript || this.data.alt;
  }

  @field(String)
  get date() {
    return new Date(
      this.data.year,
      this.data.month,
      this.data.day
    ).toLocaleDateString();
  }

  constructor(data) {
    this.data = data;
  }
}

@type
class XKCDQuery {
  @field(Comic)
  async comic(@arg(ID) id) {
    const url = id ? `/${id}` : "";
    const { data } = await axios.get(`https://xkcd.com${url}/info.0.json`);
    return new Comic(data);
  }

  @field(Comic)
  latest() {
    return this.comic();
  }

  @field(Comic)
  async random() {
    const { number } = await this.comic();
    return this.comic(Math.floor(Math.random() * number));
  }
}

const server = new ApolloServer({ schema: createSchema(XKCDQuery) });
server.listen().then(({ url }) => console.log(`Server ready at ${url}`));
```
