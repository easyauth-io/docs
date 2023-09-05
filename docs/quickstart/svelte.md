# Get started with Svelte

Learn how to authenticate users in a Svelte app using EasyAuth.


## 0. Create a Registered Client

Sign in to EasyAuth and create a new 'Registered Client' with redirect URI set to  `http://127.0.0.1:5173/auth/callback/easyauth` 

## 1. Create a new Svelte App

The following command creates an app in the directory named `myapp`. Change `myapp` to the directory name of your choice.

=== "npm"

    ``` bash
    npm create svelte@latest my-app
    cd my-app
    npm install
    ```

## 2. Install @auth/core and @auth/sveltekit


=== "npm"

    ``` bash
    npm install @auth/core @auth/sveltekit
    ```

=== "yarn"

    ``` bash
    yarn add @auth/core @auth/sveltekit

    ```


## 3. Set EasyAuth keys in `.env` file

Create a `.env` file. Fill in the values by logging in to your [EasyAuth dashboard](https://easyauth.io){target=_blank}.


Auth.js requires a secret to be set.

You can generate a good secret value:

On Unix systems: type `openssl rand -hex 32` in the terminal

Or

generate one online [here](https://generate-secret.vercel.app/32)


``` bash title=".env"
EASYAUTH_APP_URL=https://<your_subdomain>.app.easyauth.io
EASYAUTH_CLIENT_ID=<client_id>
EASYAUTH_CLIENT_SECRET=<client_secret>
AUTH_SECRET="your auth secret"
```

## 4. Add `src/hooks.server.js` file as follows:

``` js title="src/hooks.server.js"
import { SvelteKitAuth } from "@auth/sveltekit";
import {
  EASYAUTH_APP_URL,
  EASYAUTH_CLIENT_ID,
  EASYAUTH_CLIENT_SECRET,
} from "$env/static/private";

export const handle = SvelteKitAuth({
  providers: [
    {
      id: "easyauth",
      name: "easyauth",
      type: "oidc",
      params: { grant_type: "authorization_code" },
      issuer: EASYAUTH_APP_URL + "/tenantbackend",
      wellKnown: new URL(
        "/tenantbackend/.well-known/openid-configuration",
        EASYAUTH_APP_URL
      ).href,
      checks: ["pkce"],
      authorization: {
        params: { scope: "openid" },
      },
      profile(profile) {
        return {
          id: profile.sub,
          email: profile.sub,
        };
      },
      clientId: EASYAUTH_CLIENT_ID,
      clientSecret: EASYAUTH_CLIENT_SECRET,
    },
  ],
});

```

## 5. Add `src/routes/+layout.server.ts` file

``` js title="src/routes/+layout.server.ts"
import type { LayoutServerLoad } from "./$types"

export const load: LayoutServerLoad = async (event) => {
  return {
    session: await event.locals.getSession(),
  }
}

```
## 6. Edit `src/routes/+page.svelte` file

``` svelte title="src/routes/+page.svelte"

<script>
  import { signIn, signOut } from "@auth/sveltekit/client";
  import { page } from "$app/stores";
</script>

<h1>SvelteKit Auth Example</h1>
<p>
  {#if $page.data.session}
    <span class="signedInText">
      <small>Signed in as</small><br />
      <strong>{$page.data.session.user?.email ?? "User"}</strong>
    </span>
    <button on:click={() => signOut()} class="button">Sign out</button>
    
  {:else}
    <span class="notSignedInText">You are not signed in</span>
    <button on:click={() => signIn()}>Sign In with Easyauth</button>
  {/if}
  <span>
    <a href="/protected"> <button>go to protected route</button></a>
  </span>
</p>



```


## Protected Route Example
Create a route Proctected src/routes/protected/+page.svelte

``` svelte title="src/routes/protected/+page.svelte"
  <script>
    import { page } from "$app/stores"
  </script>
  

  {#if $page.data.session}
  <h1>Protected page</h1>
  <p>
    This is a protected content. You can access this content because you are
    signed in.
  </p>
  <p>Session expiry: {$page.data.session?.expires}</p>
  {:else}
  <h1>Access Denied</h1>
  <p>
    <a href="/auth/signin">
      You must be signed in to view this page
    </a>
  </p>
  {/if}

```


## Test the App

#### NOTE
Run the application on `127.0.0.1`. Then open the URL [http://127.0.0.1:5173](http://127.0.0.1:5173){target=_blank}.

=== "npm"

    ``` bash
    npm run dev
    ```

=== "yarn"

    ``` bash
    yarn dev

    ```
