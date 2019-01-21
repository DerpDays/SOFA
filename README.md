# Sofa

[![npm version](https://badge.fury.io/js/sofa-api.svg)](https://npmjs.com/package/sofa-api)
[![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)
[![renovate-app badge](https://img.shields.io/badge/renovate-app-blue.svg)](https://renovateapp.com/)

The best way to create REST APIs (is GraphQL).

## Installation

    yarn add sofa-api
    # or
    npm install sofa-api

## Getting Started

The most basic example possible:

```ts
import sofa from 'sofa-api';
import express from 'express';

const app = express();

app.use(
  '/api',
  sofa({
    schema,
  })
);

// GET /api/users
// GET /api/messages
```

## How it works

Sofa takes your GraphQL Schema, looks for available queries, mutations and subscriptions and turns all of that into REST API.

Given the following schema:

```graphql
type User {
  id: ID
  name: String
}

type Query {
  chat(id: ID): Chat
  chats: [Chat]
  me: Chat
}
```

Routes that are being generated:

```
GET /chat/:id
GET /chats
GET /me
```

### Nested data and idea behind Models

Sofa treats some types differently than others, those are called Models.

The idea behind Models is to not expose full objects in every response, especially if it's a nested, not first-level data.

For example, when fetching a list of chats you don't want to include all messages in the response, you want them to be just IDs (or links). Those messages would have to have their own endpoint. We call this type of data, a Model. In REST you probably call them Resources.

In order to treat particular types as Models you need to provide two queries, one that exposes a list (with no non-optional arguments) and the other to fetch a single entity (id field as an argument). The model itself has to have an `id` field. Those are the only requirements.

```graphql
# Message is treated as a Model
type Query {
  messages: [Message]
  message(id: ID): Message
}

type Message {
  id: ID
  # other fields ...
}
```

### Provide a Context

In order for Sofa to resolve operations based on a Context, you need te be able to provide some. Here's how you do it:

```ts
api.use(
  '/api',
  sofa({
    schema,
    async context({ req }) {
      return {
        req,
        ...yourContext,
      };
    },
  })
);
```

> You can pass a plain object or a function.

### Use full responses instead of IDs

There are some cases where sending a full object makes more sense than passing only the ID. Sofa allows you to easily define where to ignore the default behavior:

```ts
api.use(
  '/api',
  sofa({
    schema,
    ignore: ['Message.author'],
  })
);
```

Whenever Sofa tries to resolve an author of a message, instead of exposing an ID it will pass whole data.

> Pattern is easy: `Type:field` or `Type`

### Custom execute phase

By default, Sofa uses `graphql` function from `graphql-js` to turn an operation into data but it's very straightforward to pass your own logic. Thanks to that you can even use a remote GraphQL Server (with Fetch or through Apollo Links).

```ts
api.use(
  '/api',
  sofa({
    schema,
    async execute(args) {
      return yourOwnLogicHere(args);
    },
  })
);
```

### Subscriptions as webhooks

Sofa enables you to run GraphQL Subscriptions through WebHooks. It has a special API to start, update and stop a subscription.

- `POST /webhook` - stars a subscription
- `DELETE /webhook/:id` - stops it
- `POST /webhook/:id`- updats it

#### Starting a subscription

To start a new subscription you need to include following data in request's body:

- `subscription` - subscription's name, matches the name in GraphQL Schema
- `variables` - variables passed to run a subscription (optional)
- `url` - an url of your webhook receiving endpoint

After sending it to `POST /webhook` you're going to get in return a unique ID that is your started subscription's identifier.

```json
{
  "id": "SUBSCRIPTION-UNIQUE-ID"
}
```

#### Stoping a subscription

In order to stop a subscription, you need to pass its id and hit `DELETE /webhook/:id`.

#### Updating a subscription

Updating a subscription looks very similar to how you start one. Your request's body should contain:

- `variables` - variables passed to run a subscription (optional)

After sending it to `POST /webhook/:id` you're going to get in return a new ID:

```json
{
  "id": "SUBSCRIPTION-UNIQUE-ID"
}
```

#### Example

Given the following schema:

```graphql
type Subscription {
  onBook: Book
}
```

Let's start a subscription by sending that to `POST /webhook`:

```json
{
  "subscription": "onBook",
  "variables": {},
  "url": "https://app.com/new-book"
}
```

In return we get an `id` that we later on use to stop or update subscription:

    DELETE /webhook/:id

### OpenAPI and Swagger

Thanks to GraphQL's Type System Sofa is able to generate OpenAPI (Swagger) definitions out of it. Possibilities are endless here. You get all the information you need in order to write your own definitions or create a plugin that follows any specification.

```ts
import sofa, { OpenAPI } from 'sofa-api';

const openApi = OpenAPI({
  schema,
  info: {
    title: 'Example API',
    version: '3.0.0',
  },
});

app.use(
  '/api',
  sofa({
    schema,
    onRoute(info) {
      openApi.addRoute(info, {
        basePath: '/api',
      });
    },
  })
);

// writes every recorder route
openApi.save('./swagger.yml');
```

## License

[MIT](https://github.com/Urigo/sofa/blob/master/LICENSE) © Uri Goldshtein
