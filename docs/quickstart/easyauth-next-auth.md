# Get started with Next-Auth and EasyAuth

Learn how to authenticate users through NextAuth using EasyAuth provider.

???+ abstract "TLDR: Try the sample Easyauth NextAuth project"

    1. Sign in to [EasyAuth](https://easyauth.io){target=_blank} and create a new 'Registered Client' with redirect URI set to `http://127.0.0.1:3000/auth/easyauth/callback`.

    2. Clone the sample app.

        `git clone https://github.com/easyauth-io/easyauth-next-auth-example.git`

    3. Copy `.env` into `.env.local`

        `cp .env .env.local`

    4. Edit `.env.local` and set the parameters from your 'Registered Client' that you created in Step 1.

    5. Run `npm install` followed by `npm run dev`

    6. Visit [http://127.0.0.1:3000](http://127.0.0.1:3000){target=\_blank}

       Note: Replace the port `3000` in `.env.local` and redirect URI with the corresponding port where your app is runing.

## 1. Create a new Next.js Application

The following command creates a Next.js app. Configure the prompts as required.

=== "npm"

    ``` bash
    npx create-next-app@latest
    ```

=== "yarn"

    ``` bash
    yarn create next-app
    ```

## 2. Install @easyauth.io/easyauth-next-auth & next-auth

=== "npm"

    ``` bash
    npm install @easyauth.io/easyauth-next-auth next-auth --save
    ```

=== "yarn"

    ``` bash
    yarn add @easyauth.io/easyauth-next-auth next-auth
    ```

## 3. Set EasyAuth keys in `.env.local` file

Create a `.env.local` file. Fill in the values by logging in to your [EasyAuth dashboard](https://easyauth.io){target=\_blank}.

The `NEXTAUTH_URL` will be the url where your app will be running. In this case, for development purpose it will be running on `http://127.0.0.1:3000`.

```bash title=".env.local"
NEXTAUTH_URL=http://127.0.0.1:3000
EASYAUTH_CLIENT_ID=<client_id>
EASYAUTH_CLIENT_SECRET=<client_secret>
NEXT_PUBLIC_EASYAUTH_TENANT_URL=<tenant_url>
```

## 4. Create `[...nextauth].js` file & setup next-auth

Refer the [next-auth documentation](https://next-auth.js.org/getting-started/example) and create `[...nextauth].js` file in `pages/api/auth` directory. Setup next-auth as follows.

```js title="[...nextauth].js"
import NextAuth from "next-auth";
import EasyAuth from "@easyauth.io/easyauth-next-auth";

export const authOptions = {
  providers: [
    EasyAuth({
      clientId: process.env.EASYAUTH_CLIENT_ID,
      clientSecret: process.env.EASYAUTH_CLIENT_SECRET,
      tenantURL: process.env.NEXT_PUBLIC_EASYAUTH_TENANT_URL,
    }),
  ],
  secret: "my-secret",
  callbacks: {
    async jwt({token, account, profile}) {
      //Refresh token implementation
      if (account) {
        return {
          access_token: account.access_token,
          expires_at: account.expires_at,
          refresh_token: account.refresh_token,
          ...token,
        };
      } else if (Date.now() < token.expires_at * 1000) {
        return token;
      } else {
        try {
          const response = await fetch(
            new URL(
              "/tenantbackend/oauth2/token",
              process.env.NEXT_PUBLIC_EASYAUTH_TENANT_URL
            ),
            {
              headers: {"Content-Type": "application/x-www-form-urlencoded"},
              body: new URLSearchParams({
                client_id: process.env.EASYAUTH_CLIENT_ID,
                client_secret: process.env.EASYAUTH_CLIENT_SECRET,
                grant_type: "refresh_token",
                refresh_token: token.refresh_token,
              }),
              method: "POST",
            }
          );

          const tokens = await response.json();

          if (!response.ok) throw tokens;

          return {
            ...token, // Keep the previous token properties
            access_token: tokens.access_token,
            expires_at: Math.floor(Date.now() / 1000 + tokens.expires_in),
            refresh_token: tokens.refresh_token ?? token.refresh_token,
          };
        } catch (error) {
          console.error("Error refreshing access token", error);
          return {...token, error: "RefreshAccessTokenError"};
        }
      }
    },
    async session({session, token}) {
      session.user.access_token = token.access_token;
      session.error = token.error;
      return session;
    },
  },
};
export default NextAuth(authOptions);
```

## 5. Create `util/checkRefreshTokenError.js` file

This is to resolve error in refreshing Access token occured at jwt callback above. The error is generally due to expired Refresh token.

```jsx title="checkRefreshTokenError.js"
import {signOut, useSession} from "next-auth/react";
import {useEffect} from "react";

export function checkRefreshTokenError() {
  const {data: session} = useSession();

  useEffect(() => {
    if (session?.error === "RefreshAccessTokenError") {
      // Force sign out to resolve error
      // Generally the error is due to expired Refresh token.
      signOut({redirect: false}).then(() => {
        // To signout from EasyAuth Tenant.
        window.location.assign(
          `${process.env.NEXT_PUBLIC_EASYAUTH_TENANT_URL}/logout?target=${btoa(
            window.location.href
          )}`
        );
      });
    }
  }, [session]);
}
```

## 6. Edit `pages/index.js` file

```jsx title="index.js"
import Head from "next/head";
import {useSession, signIn, signOut} from "next-auth/react";
import {checkRefreshTokenError} from "@/util/checkRefreshTokenError";
import Link from "next/link";

export default function Home() {
  const {data: session, status} = useSession();
  checkRefreshTokenError();

  return (
    <>
      <Head>
        <title>EasyAuth NextAuth example</title>
        <meta name="description" content="Generated by create next app" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
      </Head>
      <main>
        <div className="hero">
          {status !== "loading" && (
            <>
              {session ? (
                <>
                  Signed in as {session.user.email} <br />
                  <button
                    onClick={() => {
                      signOut({redirect: false}).then(() => {
                        // To signout from EasyAuth Tenant.
                        window.location.assign(
                          `${
                            process.env.NEXT_PUBLIC_EASYAUTH_TENANT_URL
                          }/logout?target=${btoa(window.location.href)}`
                        );
                      });
                    }}
                  >
                    Sign out
                  </button>
                  <Link href={"/profile"}>
                    <button>EasyAuth user profile</button>
                  </Link>
                </>
              ) : (
                <>
                  Not signed in . <br />
                  <button onClick={() => signIn("easyauth")}>Sign in</button>
                </>
              )}
              <Link href={"/protected"}>
                <button>Protected page</button>
              </Link>
            </>
          )}
        </div>
      </main>
    </>
  );
}
```

## 7. Create `pages/protected.js` file

This page is protected from unauthenticated users.

```jsx title="protected.js"
import {useSession} from "next-auth/react";
import {useRouter} from "next/router";
import {checkRefreshTokenError} from "@/util/checkRefreshTokenError";
import Link from "next/link";

export default function ProtectedPage() {
  const router = useRouter();
  const {data: session, status} = useSession({
    required: true,
    onUnauthenticated() {
      router.push("/");
    },
  });

  checkRefreshTokenError();

  if (status === "loading") {
    return null;
  }

  return (
    <div className="hero">
      <h1>Protected Page</h1>
      <p>Welcome, {JSON.stringify(session.user.email)}</p>
      <Link href={"/"}>
        <button>Home page</button>
      </Link>
    </div>
  );
}
```

## 8. Create `pages/profile.js` file

Page for fetching EasyAuth profile details.

```jsx title="profile.js"
import {useSession} from "next-auth/react";
import {useRouter} from "next/router";
import React, {useEffect, useState} from "react";
import {checkRefreshTokenError} from "@/util/checkRefreshTokenError";
import Link from "next/link";

export default function Profile() {
  const router = useRouter();
  const {data: session, status} = useSession({
    required: true,
    onUnauthenticated() {
      router.push("/");
    },
  });
  const [easyAuthProfile, setEasyAuthProfile] = useState(null);

  checkRefreshTokenError();

  useEffect(() => {
    if (session) {
      fetch(
        new URL(
          "/tenantbackend/api/profile",
          process.env.NEXT_PUBLIC_EASYAUTH_TENANT_URL
        ),
        {
          method: "GET",
          headers: {
            Authorization: `Bearer ${session.user.access_token}`,
          },
        }
      )
        .then((res) => res.json())
        .then((data) => setEasyAuthProfile(data))
        .catch((err) => console.error("Failed to fetch EasyAuth profile"));
    }
  }, [status]);

  if (status === "loading") {
    return null;
  }

  return (
    <div className="hero">
      <h3>EasyAuth profile</h3>
      <p>{easyAuthProfile && JSON.stringify(easyAuthProfile)}</p>
      <Link href={"/"}>
        <button>Home page</button>
      </Link>
    </div>
  );
}
```

## 9. Add some CSS

```css title="globals.css"
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 5em 0;
  gap: 2rem;
  font-size: 1.3rem;
}

.hero button {
  cursor: pointer;
  padding: 10px 20px;
  border: none;
  background-color: rgb(34, 139, 230);
  color: white;
}

.hero button:hover {
  background-color: rgb(21, 110, 188);
}
```

## Test the App

Run the application and then open the URL [http://127.0.0.1:3000](http://127.0.0.1:3000){target=\_blank}.

=== "npm"

    ``` bash
    npm run dev
    ```

=== "yarn"

    ```bash
    yarn dev
    ```
