---
title: Run Shopify App without the App Bridge
date: 2024-10-07
description: Let's get rid of Shopify's app bridge. 
tags: shopify
author: nullndr
---

# Run Shopify App without the App Bridge

This post wants to explain how to run a Shopify'app outside the [cli](https://github.com/Shopify/cli) and the [app bridge](https://shopify.dev/docs/api/app-bridge).

> Please pay attention, you won't be able to use anything from the App Bridge API. The installation process is also affected, since you won't be able to
> install the app without it.

Let's start with the [Shopify Remix Template](https://github.com/Shopify/shopify-app-template-remix) setup, one of the main file is the `shopify.server.ts`:

```ts app/shopify.server.ts
import "@shopify/shopify-app-remix/adapters/node";
import {
  ApiVersion,
  AppDistribution,
  shopifyApp,
} from "@shopify/shopify-app-remix/server";
import { PrismaSessionStorage } from "@shopify/shopify-app-session-storage-prisma";
import { restResources } from "@shopify/shopify-api/rest/admin/2024-07";
import prisma from "./db.server";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY,
  apiSecretKey: process.env.SHOPIFY_API_SECRET || "",
  apiVersion: ApiVersion.October24,
  scopes: process.env.SCOPES?.split(","),
  appUrl: process.env.SHOPIFY_APP_URL || "",
  authPathPrefix: "/auth",
  sessionStorage: new PrismaSessionStorage(prisma),
  distribution: AppDistribution.AppStore,
  restResources,
  future: {
    unstable_newEmbeddedAuthStrategy: true,
  },
  ...(process.env.SHOP_CUSTOM_DOMAIN
    ? { customShopDomains: [process.env.SHOP_CUSTOM_DOMAIN] }
    : {}),
});

export default shopify;
export const apiVersion = ApiVersion.October24;
export const addDocumentResponseHeaders = shopify.addDocumentResponseHeaders;
export const authenticate = shopify.authenticate;
export const unauthenticated = shopify.unauthenticated;
export const login = shopify.login;
export const registerWebhooks = shopify.registerWebhooks;
export const sessionStorage = shopify.sessionStorage;
```

It exports a bunch of utilities, but the one we are interested in is the `authenticate` one. It has a bunch of methods to check if a request is from Shopify Admin, Shopify Flow, a Shopify Webhook etc.

You should use it as the first call of each `loader` and `action` in all your routes:

```ts app/routes/app.tsx
export const loader = async ({ request }: LoaderFunctionArgs) => {
  await authenticate.admin(request);

  return "This request is from Shopify Admin!";
};
```

> A great convention you can use is to run your embedded app within the /app layout, in this way you can configure common react contexts to be shared across the app. Also,
> you may like to have the /webhook route on the first level, not something like /app/webhook 

The problem with `await authenticate.admin(request)` is that it throws a redirect response to the `authPathPrefix` you defined above, so we simply need the call to not throw.

The solution is quite easy, we need to encapsulate the logic for it, returning one thing from the session when we want to run the app outside the app bridge.

I said "one thing from the session" because it is unlikely you are using the prisma [Session](https://github.com/Shopify/shopify-app-template-remix/blob/main/prisma/schema.prisma) model to save
your business logic, I bet you have a `Shop` model where you save your data, maybe from the `afterAuth` hook.

If that's the case excellent, you may have something like the following:

```prisma prisma/schema.prisma
model Session {
  id            String    @id
  shop          String
  state         String
  isOnline      Boolean   @default(false)
  scope         String?
  expires       DateTime?
  accessToken   String
  userId        BigInt?
  firstName     String?
  lastName      String?
  email         String?
  accountOwner  Boolean   @default(false)
  locale        String?
  collaborator  Boolean?  @default(false)
  emailVerified Boolean?  @default(false)
}

model Shop {
  id              String  @id @default(cuid())
  shopifyDomain   String  @unique
  accessToken     String
}
```

Do you see the point here? We actually do not need the session if we already have the `shopifyDomain` (It is the one in the form `<name>.myshopify.com`, you can find it in the `Session.shop` column).

Let's write now some logic to handle this, we want to run our app with a `shopifyDomain` we define, something like:

```ts
export async function requireShopifyDomain(request: Request) {
  if(process.env.NODE_ENV === "development" && process.env.RUN_AS_SHOPIFY_DOMAIN) {
    const shopifyDomain = process.env.RUN_AS_SHOPIFY_DOMAIN;
    return shopifyDomain;
  }

  const { session } = await authenticate.admin(request);
  return session.shop;
}
```

Excellent, we can retrieve now the `shopifyDomain` we define as an env, just in development to avoid any potential issue while running in production.

Let's write now a simple utility to handle all loaders requests in the same way:

```ts
export function handleLoaderRequest<T>(
  request: Request,
  callback: (shopifyDomain: string) => Promise<T>
) {
  const shopifyDomain = await requireShopifyDomain(request);
  return await callback(shopifyDomain);
}
```

With this our `loader` gets updated like the following:

```ts app/routes/app.tsx
export const loader = async ({ request }: LoaderFunctionArgs) => {
  return handleLoaderRequest(request, async (shopifyDomain) => {
    return "This request is from Shopify Admin... probably.";
  });
};
```

You may now ask yourself: "Great, but I have no way to access the GraphQL admin". You are right, it is not possible to access it in this way, but the Shopify GraphQL client is not the only client out here.

A great replacement for it is [Genql](https://github.com/remorses/genql), to build the client for the shop we only need two things: the shopify domain and the access token.

So let's start with a little model to get the info about the shop:

```ts app/models/shop.ts
export async function findShop(shopifyDomain: string) {
  return prisma.shop.findUniqueOrThrown({
    where: {
      shopifyDomain,
    },
  });
} 
```

After this let's set up the GraphQL schema for Genql, simply run the following command:

```sh
npx genql -S --output "app/lib/genql/generated.server" \ 
  --endpoint "https://<name>.myshopify.com/admin/api/2024-07/graphql.json" \ 
  -H "X-Shopify-Access-Token: <access token>" --esm
```

Replace the `<name>.myshopify.com` and `<access token>` with some real data (you can also use the data from a test store, the command is just needed to generate the graphql schema).

Let's now create the little utility we will use to generate the graphql client:

```ts app/lib/genql/index.ts
import { createClient } from "./generated.server";
export * from "./generated.server";

type CreateGenqlClientArgs = {
  shopifyDomain: string;
  accessToken: string;
};

export function createGenqlClient({
  shopifyDomain,
  accessToken,
}: CreateGenqlClientArgs) {
  return createClient({
    url: `https://${shopifyDomain}/admin/api/2024-07/graphql.json`,
    headers: {
      "X-Shopify-Access-Token": accessToken,
    },
  });
}
```

That's it, let's glue all together:

```ts app/routes/app.tsx
export const loader = async ({ request }: LoaderFunctionArgs) => {
  return handleLoaderRequest(request, async (shopifyDomain) => {
    const shop = await findShop(shopifyDomain);
    const genqlClient = createGenqlClient(shop);
    const queryResult = await genqlClient.query({
      shop: {
        id: true,
      },
    });
    return `This request is from shop ${queryResult.shop.id}.`;
  });
};
```

Nice, now time for some questions.

## What about the billing api?

The `authenticate.admin()` call also returns the `billing` object, that allows you to handle all aspects of your app's billing,
but it is just a wrapper for some graphql mutations, like [appPurchaseOneTimeCreate](https://shopify.dev/docs/api/admin-graphql/2024-10/mutations/apppurchaseonetimecreate) and
[appSubscriptionCreate](https://shopify.dev/docs/api/admin-graphql/latest/mutations/appsubscriptioncreate), you can handle them with Genql as well.

## I also use some actions in my app!

The process to handle `action`s request is the same, but you just need to check for the method of the request:

```typescript
type PostHandler<T> = {
  POST: (shopifyDomain: string) => Promise<T>;
};

type PutHandler<T> = {
  PUT: (shopifyDomain: string) => Promise<T>;
};

type PatchHandler<T> = {
  PATCH: (shopifyDomain: string) => Promise<T>;
};

type DeleteHandler<T> = {
  DELETE: (shopifyDomain: string) => Promise<T>;
};

type RequestHandlerMap<PostResult, PutResult, PatchResult, DeleteResult> =
  Partial<
    PostHandler<PostResult> &
      PutHandler<PutResult> &
      PatchHandler<PatchResult> &
      DeleteHandler<DeleteResult>
  >;

export async function handleActionRequest<
  PostResult = never,
  PutResult = never,
  PatchResult = never,
  DeleteResult = never,
>(
  request: Request,
  map: RequestHandlerMap<PostResult, PutResult, PatchResult, DeleteResult>,
) {
  const requestMethod = request.method as keyof typeof map;
  const requestHandler = map[requestMethod];

  if (requestHandler) {
    const shopifyDomain = await requireShopifyDomain(request);
    return await requestHandler(shopifyDomain);
  }

  throw methodNotAllowed();
}
```

You can update all your `action`s like this:


```ts
export const action = async ({ request }: ActionFunctionArgs) => {
  return handleActionRequest(request, {
    POST: async (shopifyDomain) => {
      // logic for the POST method
    },
    DELETE: async (shopifyDomain) => {
      // logic for the DELETE method
    },
  });
}
```

## Do I need the record of the shop to be in my db?

Yes, since it is not possible to use the app bridge to get the session the only possible point is your db.