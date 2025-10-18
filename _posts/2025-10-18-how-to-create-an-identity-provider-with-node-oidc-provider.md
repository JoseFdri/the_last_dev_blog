---
layout: post
title: "How to Create an Identity Provider with Node OIDC Provider"
date: 2025-10-17 10:00:00 -0500
categories: oidc
author: Jose Rodriguez
---

In this article, we will learn how to create our own Identity Provider (IdP) using OpenID Connect (OIDC). We will use the popular `node-oidc-provider` library to build our IdP.

## What is an Identity Provider?

An Identity Provider is a service that manages identity information and provides authentication services. When you log in to an application using your Google or GitHub account, you are using them as Identity Providers.

## What is OpenID Connect?

OIDC is a simple identity layer on top of the OAuth 2.0 protocol. It allows clients to verify the identity of the end-user based on the authentication performed by an Authorization Server, as well as to obtain basic profile information about the end-user in an interoperable and REST-like manner.

## Part 1: Building the Identity Provider

### Setting up the IdP project

First, let's create a new Node.js project for our IdP:

```bash
mkdir my-oidc-server
cd my-oidc-server
npm init -y
npm install oidc-provider express
```

`oidc-provider` is an ECMAScript module, so we need to enable it by adding `"type": "module"` to our `package.json` file.

```json
{
  "name": "my-oidc-server",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "start": "node index.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2",
    "oidc-provider": "^8.4.5"
  }
}
```

### Basic OIDC Provider

Now, let's create our basic OIDC provider. Create an `index.js` file with the following content:

```javascript
import express from 'express';
import { Provider } from 'oidc-provider';

const app = express();
const port = 3000;
const issuer = `http://localhost:${port}`;

const configuration = {
  clients: [
    {
      client_id: 'my-app',
      client_secret: 'my-secret',
      redirect_uris: ['http://localhost:3001/cb'],
      response_types: ['code'],
      grant_types: ['authorization_code'],
    },
  ],
  pkce: {
    required: () => true,
    methods: ['S256'],
  },
  findAccount: async (ctx, id) => {
    // This is an example of how to find an account. In a real-world scenario, you would use a database.
    const users = [
      {
        accountId: 'user1',
        claims: (use, scope, claims, rejected) => {
          return {
            sub: 'user1',
            email: 'user1@example.com',
            email_verified: true,
          };
        },
      },
    ];

    const user = users.find((user) => user.accountId === id);

    if (!user) {
      return undefined;
    }

    return user;
  },
};

const oidc = new Provider(issuer, configuration);

app.use('/oidc', oidc.callback());

oidc.listen(port, () => {
  console.log(`oidc-provider listening on port ${port}, check ${issuer}/.well-known/openid-configuration`);
});
```

### Registering a client

To allow a client application to connect to our IdP, we need to register it in the IdP's configuration. In this configuration, we have registered a client with the following properties:

-   `client_id`: A unique identifier for the client application.
-   `client_secret`: A secret that the client application will use to authenticate with the IdP.
-   `redirect_uris`: An array of URLs where the user will be redirected after they have been authenticated.
-   `response_types`: An array of response types that the client application can use. In this case, we are using the `code` response type, which is used for the Authorization Code flow.
-   `grant_types`: An array of grant types that the client application can use. In this case, we are using the `authorization_code` grant type, which is used for the Authorization Code flow.

#### Dynamic Client Registration

`node-oidc-provider` also supports Dynamic Client Registration (DCR), which allows clients to be created via an API. This is useful when you have a large number of clients or when you want to automate the client registration process.

To enable DCR, you need to add the `features.registration` property to the IdP's configuration:

```javascript
const configuration = {
  // ...
  features: {
    registration: { enabled: true },
  },
};
```

Once DCR is enabled, you can create clients by sending a POST request to the registration endpoint (`/reg`). The body of the request should be a JSON object with the client's metadata.

Here is an example of how to create a client using `curl`:

```bash
curl -X POST -H "Content-Type: application/json" -d '
{
  "client_name": "My Dynamic Client",
  "redirect_uris": ["http://localhost:3002/cb"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"]
}' http://localhost:3000/reg
```

This command sends a POST request to the registration endpoint with the client's metadata in the request body. The IdP will then create a new client and return its metadata, including the `client_id` and `client_secret`.

### Handling Interaction

When a user tries to log in, the OIDC provider will redirect them to an interaction URL. We need to handle this interaction and authenticate the user. For this example, we will automatically log in the user.

```javascript
app.use(async (req, res, next) => {
  const details = await oidc.interactionDetails(req, res);
  const { uid, prompt, params } = details;

  if (prompt.name === 'login') {
    const result = {
      login: {
        accountId: 'user1',
      },
    };
    await oidc.interactionFinished(req, res, result, { mergeWithLastSubmission: false });
  } else {
    next();
  }
});
```

## Part 2: Building the Client Application

### Setting up the client project

Now that we have our IdP, let's create a simple client application that will use it to authenticate users.

First, let's create a new Node.js project for our client app:

```bash
mkdir my-client-app
cd my-client-app
npm init -y
npm install express openid-client
```

### Client Application Code

Now, let's create an `index.js` file with the following content:

```javascript
import express from 'express';
import { Issuer } from 'openid-client';

const app = express();
const port = 3001;

const oidcIssuer = await Issuer.discover('http://localhost:3000');

const client = new oidcIssuer.Client({
  client_id: 'my-app',
  client_secret: 'my-secret',
  redirect_uris: ['http://localhost:3001/cb'],
  response_types: ['code'],
});

app.get('/', (req, res) => {
  res.send('<a href="/login">Log in</a>');
});

app.get('/login', (req, res) => {
  const url = client.authorizationUrl({
    scope: 'openid email profile',
  });
  res.redirect(url);
});

app.get('/cb', async (req, res) => {
  const params = client.callbackParams(req);
  const tokenSet = await client.callback('http://localhost:3001/cb', params);
  const userInfo = await client.userinfo(tokenSet.access_token);
  res.send(userInfo);
});

app.listen(port, () => {
  console.log(`Client application listening on port ${port}`);
});
```

## Part 3: Running the Example

Now that we have our IdP and our client application, let's see how the authentication process works:

1.  Start both the IdP and the client application in separate terminals.
2.  Open your browser and navigate to `http://localhost:3001`.
3.  Click the "Log in" link.
4.  You will be redirected to the IdP's login page. In our example, we are automatically logging in the user, so you will be redirected back to the client application.
5.  The client application will receive the authorization code and exchange it for an access token and an ID token.
6.  The client application will then use the access token to get the user's information from the userinfo endpoint.
7.  Finally, the client application will display the user's information.

## Conclusion

In this article, we have learned how to create a basic Identity Provider using `node-oidc-provider`. We have seen how to configure PKCE, register clients, and authenticate users. We have also created a simple client application to test our IdP. This is just a starting point. `node-oidc-provider` is a very powerful and flexible library that allows you to customize every aspect of your IdP.
