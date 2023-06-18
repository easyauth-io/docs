# Get started with React

Learn how to authenticate users in a React app using EasyAuth.


???+ abstract "TLDR: Try the sample React project"

    1. Sign in to [easyauth.io](https://easyauth.io){target=_blank} and create a new 'Registered Client' with redirect URI set to `http://127.0.0.1:3000`
    
    2. Clone the sample app from [https://github.com/easyauth-io/easyauth-react-example](https://github.com/easyauth-io/easyauth-react-example)

        `git clone https://github.com/easyauth-io/easyauth-react-example.git`

    3. Copy `.env` into `.env.local`

        `cp .env .env.local`

    4. Edit `.env.local` and set the parameters from your 'Registered Client' that you created in Step 1.

    5. Run `npm install` followed by `npm run start`

    6. Visit [http://127.0.0.1:3000](http://127.0.0.1:3000){target=_blank}


## 1. Create a new React App

The following command creates an app in the directory named `myapp`. Change `myapp` to the directory name of your choice.

=== "npm"

    ``` bash
    npx create-react-app myapp
    cd myapp
    ```

=== "yarn"

    ``` bash
    yarn create react-app myapp
    cd myapp
    ```

## 2. Install @easyauth.io/easyauth-react


=== "npm"

    ``` bash
    npm install @easyauth.io/easyauth-react --save
    ```

=== "yarn"

    ``` bash
    yarn add @easyauth.io/easyauth-react
    ```


## 3. Set EasyAuth keys in `.env.local` file

Create a `.env.local` file. Fill in the values by logging in to your [EasyAuth dashboard](https://easyauth.io){target=_blank}.

The `REACT_APP_EASYAUTH_REDIRECT_URL` will be the url where your app will be running. In this case, for development purpose it will be running on `http://127.0.0.1:3000`.


``` bash title=".env.local"
REACT_APP_EASYAUTH_APP_URL=https://<your_subdomain>.app.easyauth.io
REACT_APP_EASYAUTH_CLIENT_ID=<client_id>
REACT_APP_EASYAUTH_REDIRECT_URL=http://127.0.0.1:3000
```

## 4. Edit `src/index.js` file as follows:

``` js title="src/index.js"
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';
import { EasyauthProvider } from "@easyauth.io/easyauth-react";

// ================ Wrap <App> with  <EasyauthProvider> ================= //

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <EasyauthProvider>
      <App />
    </EasyauthProvider>
  </React.StrictMode>
);


// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

## 5. Edit `src/App.js` file

``` js title="src/App.js"
import logo from "./logo.svg";
import "./App.css";
import {
  SignedIn,
  SignedOut,
  UserButton,
  UserProfile,
  useEasyauth,
  useUser,
} from "@easyauth.io/easyauth-react";

function App() {
  const auth = useEasyauth();
  const { isAuthenticated, user, isLoading } = useUser();

  switch (auth.activeNavigator) {
    case "signinSilent":
      return <div>Signing you in...</div>;
    case "signoutRedirect":
      return <div>Signing you out...</div>;
  }

  if (auth.isLoading) {
    return <h1>Loading...</h1>;
  }

  if (auth.error) {
    return <div>Oops... {auth.error.message}</div>;
  }

  return (
    <>
      <SignedIn>
        <div className="App">
          <header className="App-header">
            <div className="Usr-btn">
              <UserButton />
            </div>
            <img src={logo} className="App-logo" alt="logo" />
            <p>Hello {user.email} </p>
          </header>
        </div>
      </SignedIn>
      <SignedOut>
        <div className="App">
          <header className="App-header">
            <img src={logo} className="App-logo" alt="logo" />
            <p>
              Edit <code>src/App.js</code> and save to reload.
            </p>
            <button
              style={{padding: "0.2em 1em", "fontSize":"1em", "cursor": "pointer"}}
              onClick={() => void auth.signinRedirect()}
            >
              Log in
            </button>
          </header>
        </div>
      </SignedOut>
    </>
  );
}

export default App;

```

## Test the App

Run the application on `127.0.0.1`. Then open the URL [http://127.0.0.1:3000](http://127.0.0.1:3000){target=_blank}.

=== "npm"

    ``` bash
    HOST=127.0.0.1 npm run start
    ```

=== "yarn"

    ``` bash
    HOST=127.0.0.1 yarn start
    ```
=== "npm on Windows"

    ``` bash
    set HOST=127.0.0.1
    npm run start
    ```

=== "yarn on Windows"

    ``` bash
    set HOST=127.0.0.1
    yarn start
    ```


## Next Steps

Query user information from EasyAuth is given by useUser() hook.