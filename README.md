# MicrosoftStrategy for [Remix](https://remix.run/) using [Remix-Auth](https://github.com/sergiodxa/remix-auth)

The Microsoft strategy is used to authenticate users against an account on [Microsoft Active Directory](https://docs.microsoft.com/en-us/azure/active-directory/develop/) using [Remix-Auth](https://github.com/sergiodxa/remix-auth).
This can be a work/school account or a personal Microsoft account, like Skype, Xbox and Outlook.com. It extends the OAuth2Strategy.

## Supported runtimes

| Runtime    | Has Support |
| ---------- | ----------- |
| Node.js    | ✅          |
| Cloudflare | ✅          |

## How to use

### Create an OAuth application

Follow the steps on [the Microsoft documentation](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app) to create a new App Registration. You should select **Web** as the platform, configure a **Redirect URI** and add a client secret.

    If you want to support login with both personal Microsoft accounts and school/work accounts, you might need to configure the supported account types by editing the manifest file. Set `signInAudience` value to `MicrosoftADandPersonalMicrosoftAccount` to allow login also with personal accounts.

Change your redirect URI to `https://example.com/auth/microsoft/callback` or `http://localhost:4200/auth/microsoft/callback` if you run it locally.

Be sure to copy the client secret, redirect URI, Tenant ID and the Application (client) ID (under Overview) because you will need them later.

### Install dependencies

```bash
npm install remix-auth-microsoft remix-auth remix-auth-oauth2 @remix-run/server-runtime
```

### Create the strategy instance

```ts
// app/services/auth.server.ts
import { MicrosoftStrategy } from "remix-auth-microsoft";
import { Authenticator } from "remix-auth";
import { sessionStorage } from "~/services/session.server";

export let authenticator = new Authenticator<User>(sessionStorage); //User is a custom user types you can define as you want

let microsoftStrategy = new MicrosoftStrategy(
  {
    clientId: "YOUR_CLIENT_ID",
    clientSecret: "YOUR_CLIENT_SECRET",
    redirectUri: "https://example.com/auth/microsoft/callback",
    tenantId: "YOUR_TENANT_ID", // optional - necessary for organization without multitenant (see below)
    scope: "openid profile email", // optional
    prompt: "login", // optional
  },
  async ({ accessToken, extraParams, profile }) => {
    // Here you can fetch the user from database or return a user object based on profile
    // return {profile}
    // The returned object is stored in the session storage you are using by the authenticator

    // If you're using cookieSessionStorage, be aware that cookies have a size limit of 4kb
    // For example this won't work
    // return {accessToken, extraParams, profile}
    return User.findOrCreate({ email: profile.emails[0].value });
  }
);

authenticator.use(microsoftStrategy);
```

See [Microsoft docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow) for more information on `scope` and `prompt` parameters.

### Applications with single-tenant authentication (no multitenant allowed)

If you want to allow login only for users from a single organization, you should add the `tenantId` attribute to the configuration passed to `MicrosoftStrategy`.
The value of `tenantId` should be the **Directory (tenant) ID** found under **Overview** in your App Registration page.

You must also select **Accounts in this organizational directory** as Supported account types in your App Registration.

### Setup your routes

```tsx
// app/routes/login.tsx
export default function Login() {
  return (
    <form action="/auth/microsoft" method="post">
      <button>Login with Microsoft</button>
    </form>
  );
}
```

```tsx
// app/routes/auth/microsoft.tsx
import type { ActionFunction, LoaderFunction } from "@remix-run/node";
import { authenticator } from "~/auth.server";
import { redirect } from "@remix-run/node";

export const loader: LoaderFunction = () => redirect("/login");

export const action: ActionFunction = ({ request }) => {
  return authenticator.authenticate("microsoft", request);
};
```

```ts
// app/routes/auth/microsoft/callback.tsx
import type { LoaderFunction } from "@remix-run/node";
import { authenticator } from "~/auth.server";

export const loader: LoaderFunction = ({ request }) => {
  return authenticator.authenticate("microsoft", request, {
    successRedirect: "/dashboard",
    failureRedirect: "/login",
  });
};
```

### Add Session Storage

```ts
// app/services/session.server.ts
import { createCookieSessionStorage } from "@remix-run/node";

export let sessionStorage = createCookieSessionStorage({
  cookie: {
    name: "_session", // use any name you want here
    sameSite: "lax", // this helps with CSRF
    path: "/", // remember to add this so the cookie will work in all routes
    httpOnly: true, // for security reasons, make this cookie http only
    secrets: ["s3cr3t"], // replace this with an actual secret
    secure: process.env.NODE_ENV === "production", // enable this in prod only
  },
});

export let { getSession, commitSession, destroySession } = sessionStorage;
```
