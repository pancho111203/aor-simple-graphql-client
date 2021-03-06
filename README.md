# aor-simple-graphql-client

[![Build Status](https://travis-ci.org/marmelab/aor-simple-graphql-client.svg?branch=master)](https://travis-ci.org/marmelab/aor-simple-graphql-client)

A simple GraphQL client for [admin-on-rest](https://github.com/marmelab/admin-on-rest/)
built with [Apollo](http://www.apollodata.com/)

## About GraphQL and Apollo

This library is meant to be used with Apollo on the **client** side but
you're free to use any graphql server.

## Installation

Install with:

```sh
npm install --save aor-simple-graphql-client
```

or

```sh
yarn add aor-simple-graphql-client
```

## Usage

Let's create a file for our admin page `admin.js`:

```js
import React, { Component } from 'react';
import { buildApolloClient } from 'aor-simple-graphql-client';

import { Admin, Resource } from 'admin-on-rest';
import { Delete } from 'admin-on-rest/lib/mui';

import createRestClient from '../lib/admin-on-rest/client';

import { PostCreate, PostEdit, PostList } from '../components/admin/posts';

const client = new ApolloClient();

class AdminPage extends Component {
    constructor() {
        super();
        this.state = { restClient: null };
    }
    componentDidMount() {
        buildApolloClient()
            .then(restClient => this.setState({ restClient }));
    }

    render() {
        const { restClient } = this.state;

        if (!restClient) {
            return <div>Loading</div>;
        }

        return (
            <Admin restClient={restClient}>
                <Resource name="Post" list={PostList} edit={PostEdit} create={PostCreate} remove={Delete} />
            </Admin>
        );
    }
}

export default AdminPage;
```

And that's it, `buildApolloClient` will create a default ApolloClient for you and
run an introspection query on your graphql endpoint.

By default, it expect the following queries and mutations for each resource:

### List resources with pagination

Example with resource `Post`:

```graphql
getPageOfPosts(page: Int, perPage: Int, sortField: String, sortOrder: String, filter: String) {
    items: [Post]
    totalCount: Int
}
```

Note that the function should be named with the plural version of `Post`.
We use [pluralize](https://github.com/blakeembrey/pluralize) to generate it.

`filter` may contain a serialized JSON object, for example:

```js
'{ "authorId": "4e80878c-6baa-4506-a93c-ef99b74e73e0" }'
```

### Get a resource

Example with resource `Post`:

```graphql
getPost(id: ID!) Post
```

### Create a new resource

Example with resource `Post`:

```graphql
createPost(data: String) Post
```

`data` is a serialized JSON object, for example:

```js
'{ "title": "My first post", "authorId": "4e80878c-6baa-4506-a93c-ef99b74e73e0", "body": "..." }'
```

### Update a resource

Example with resource `Post`:

```graphql
updatePost(data: String) Post
```

`data` is a serialized JSON object, for example:

```js
'{ "id": "c02e92e8-2a21-4ae7-9197-cb9601861a44", "title": "My first post", "authorId": "4e80878c-6baa-4506-a93c-ef99b74e73e0", "body": "..." }'
```

### Remove a resource

Example with resource `Post`:

```graphql
removePost(id: ID!) Boolean
```

## Options

### Customize the Apollo client

You can either supply the client options by calling `buildApolloClient` like this:

```js
buildApolloClient({ clientOptions: { url: 'http://localhost:3000', ...otherOptions } });
```

Or supply your client directly with:

```js
buildApolloClient({ client: myClient });
```

### Customize the introspection

These are the default options for introspection:

```js
const introspectionOptions = {
    url: null, // The url of the GraphQL endpoint, if not supplied, will fall back on the client network interface url
    includeTypes: null, // Either an array of types to include or a function which will be called with each OBJECT type discovered through introspection
    excludeTypes: null, // Either an array of types to exclude or a function which will be called with each OBJECT type discovered through introspection (`Query` and `Mutation` are excluded anyway)
    includeQueries: null, // Either an array of queries to include or a function which will be called with each query discovered through introspection
    excludeQueries: null, // Either an array of queries to exclude or a function which will be called with each query discovered through introspection
    includeMutations: null, // Either an array of mutations to include or a function which will be called with each mutation discovered through introspection
    excludeMutations: null, // Either an array of mutations to exclude or a function which will be called with each mutation discovered through introspection
    excludeFields: null, // Either an array of fields to exclude or a function which will be called with each field discovered through introspection on a specific object (more details below)

    // This contains templates for defining the queries and mutations names which will also be used as the operations names
    templates: {
        GET_LIST: resourceType => `getPageOf${pluralize(resourceType.name)}`,
        GET_ONE: resourceType => `get${resourceType.name}`,
        CREATE: resourceType => `create${resourceType.name}`,
        UPDATE: resourceType => `update${resourceType.name}`,
        DELETE: resourceType => `remove${resourceType.name}`,
    },
}
```

And how you pass them to the `buildApolloClient` function:

```js
buildApolloClient({ introspection: introspectionOptions });
```

Note that `excludeXXX` and `includeXXX` are mutualy exclusives and
that `includeXXX` will always take precendance.

`excludeFields` deserves more details. If supplying a function, it
will receive the following parameters:

- `field`: the field definition (see the documentation on
    [introspection](http://graphql.org/learn/introspection/) for more details)
- `resource`: the resource type (for example: `Post`)
- `type`: the operation type (matching those of **admin-on-rest**, for example: `GET_LIST`)

### Supply your own queries and mutations

You need more control? Then provide your queries with the following format:

```js
const queries = {
    Post: {
        GET_LIST: () => gql`your query`, // Variables will be: { page: Int, perPage: Int, sortFilter: String, sortOrder: String, filter: String }
        GET_MANY: () => gql`your query`, // Optional, see note below. Variables will be: { filter: String }
        GET_MANY_REFERENCE: () => gql`your query`, // Optional, see note below. Variables will be: { filter: String }
        GET_ONE: () => gql`your query`, // Variables will be: { id: ID }
        CREATE: () => gql`your query`, // Variables will be: { data: String }
        UPDATE: () => gql`your query`, // Variables will be: { data: String }
        DELETE: () => gql`your query`, // Variables will be: { id: ID }
    }
}
```

**Note**: `GET_MANY` and `GET_MANY_REFERENCE` are optional.
If not specified, `GET_LIST` will be called with the `filter` and `perPage` set to `1000`.

## Realtime updates

Using ApolloClient, one is able to get real time updates when data changes:
see their [documentation](http://dev.apollodata.com/core/apollo-client-api.html#ApolloClient\.watchQuery) for details.

With `aor-simple-graphql-client`, you can enable real time updates like this:

```js
import React, { Component } from 'react';
import { buildApolloClient } from 'aor-simple-graphql-client';

import { Admin, Resource } from 'admin-on-rest';
import { Delete } from 'admin-on-rest/lib/mui';

import createRestClient from '../lib/admin-on-rest/client';

import { PostCreate, PostEdit, PostList } from '../components/admin/posts';

const client = new ApolloClient();

class AdminPage extends Component {
    constructor() {
        super();
        this.state = { restClient: null };
    }
    componentDidMount() {
        buildApolloClient()
            .then(restClient => this.setState({ restClient }));
    }

    render() {
        const { restClient } = this.state;

        if (!restClient) {
            return <div>Loading</div>;
        }

        return (
            <Admin restClient={restClient} customSagas={[restClient.saga()]}>
                <Resource name="Post" list={PostList} edit={PostEdit} create={PostCreate} remove={Delete} />
            </Admin>
        );
    }
}

export default AdminPage;
```

We simply pass a custom [saga](https://marmelab.com/admin-on-rest/AdminResource.html#customsagas)
to the `Admin` component.

By default, it will use the Apollo polling mechanism with a `pollInterval` of 2 seconds for all
`GET_LIST` and `GET_ONE` requests.

### Customization

You can specify the options to pass to the [watchQuery](http://dev.apollodata.com/core/apollo-client-api.html#ApolloClient\.watchQuery)
function.

#### For all resources and request types

```js
    const apolloWatchOptions = {
        pollInterval: 10000,
    };

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

Or, you can supply a function instead of an object and it will be call
with `resource` and `requestType` parameters:

```js
    const apolloWatchOptions = (resource, requestType) => ({
        pollInterval: 10000,
    });

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

#### For a specific resources and all request types

```js
    const apolloWatchOptions = {
        Post: {
            pollInterval: 10000,
        },
        Comment: {
            pollInterval: 5000,
        },
    };

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

Or, you can supply a function instead of an object and it will be call
with a `requestType` parameter:

```js
    const apolloWatchOptions = (resource, requestType) => {
        Post: {
            pollInterval: 10000,
        },
        Comment: {
            (requestType): => ({
                pollInterval: 5000,
            }),
        },
    };

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

#### For a specific resources and request type

```js
    const apolloWatchOptions = {
        Post: {
            pollInterval: 2000,
        },
        Comment: {
            GET_ONE: {
                pollInterval: 10000,
            },
            GET_LIST: {
                pollInterval: 5000,
            },
        },
    };

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

Or, you can supply a function instead of an object:

```js
    const apolloWatchOptions = (resource, requestType) => {
        Post: {
            pollInterval: 10000,
        },
        Comment: {
            GET_ONE: {
                pollInterval: 10000,
            },
            GET_LIST: {
                (): => ({
                    pollInterval: 5000,
                }),
            },
        },
    };

    const apolloSaga = restClient.saga(apolloWatchOptions);
```

## TODO

- Sample application

## Contributing

Coverage data is available in `./coverage` after executing `make test`.

An HTML report is generated in `./coverage/lcov-report/index.html`.
