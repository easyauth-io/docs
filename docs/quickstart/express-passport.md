# Get started with Express and Passport.js

Learn how to authenticate users through Express app and Passport.js using EasyAuth strategy.

???+ abstract "TLDR: Try the sample Easyauth Passport project"

    1. Sign in to [easyauth.io](https://easyauth.io){target=_blank} and create a new 'Registered Client' with redirect URI set to `http://127.0.0.1:3000/auth/easyauth/callback`

    2. Clone the example app from [https://github.com/easyauth-io/easyauth-passport-example](https://github.com/easyauth-io/easyauth-passport-example)

        `git clone https://github.com/easyauth-io/easyauth-passport-example`

    3. Edit `.env` and set the parameters from your 'Registered Client' that you created in Step 1

    5. Run `npm install` followed by `npm run start`

    6. Visit [http://127.0.0.1:3000/auth/easyauth/login](http://127.0.0.1:3000/auth/easyauth/login){target=_blank}

## 1. Create a new Nodejs Application

The following command creates a node app in the directory named `myapp`. Change `myapp` to the directory name of your choice.

=== "npm"

    ``` bash
    mkdir myapp
    cd myapp
    npm init
    ```

## 2. Install required Dependencies

This is to install the dependencies required to setup express app and passport.

```bash
npm install express passport express-session dotenv @easyauth.io/passport-easyauth
```

## 3. Set EasyAuth keys in `.env` file

Create a `.env` file. Log in to your [EasyAuth dashboard](https://easyauth.io){target=\_blank} and create a new Registered Client. Set
the Redirect URL to `http://127.0.0.1:3000/auth/easyauth/callback` and then fill the values as shown.

```bash title=".env"
EASYAUTH_DISCOVERY_URL=https://<your_subdomain/tenant_name>.app.easyauth.io
EASYAUTH_CLIENT_ID=<client_id>
EASYAUTH_CLIENT_SECRET=<client_secret>
PORT=3000
```

## 4. Edit `strategy.js`:

Create a new file named `strategy.js` for initializing EasyAuth strategy and edit it as follows:

```js title="strategy.js"
const passport = require('passport');
const EasyAuthStrategy = require('@easyauth.io/passport-easyauth');
require('dotenv').config();

const PORT = process.env.PORT;
passport.use(
  new EasyAuthStrategy(
    {
      discoveryURL: process.env.EASYAUTH_DISCOVERY_URL,
      clientID: process.env.EASYAUTH_CLIENT_ID,
      clientSecret: process.env.EASYAUTH_CLIENT_SECRET,
      callbackURL: [`http://127.0.0.1:${PORT}/auth/easyauth/callback`],
    },
    function (tokenset, userinfo, done) {
      done(null, userinfo);
    }
  )
);
```

## 5. Edit `index.js`:

Create a new file named `index.js` and edit it as follows:

```js title="index.js"
const express = require('express');
const session = require('express-session');
const passport = require('passport');
const app = express();

require('./strategy');
require('dotenv').config();

const PORT = process.env.PORT;

app.use(
  session({
    secret: 'your secret key',
    resave: false,
    saveUninitialized: false,
  })
);

app.use(passport.initialize());
app.use(passport.session());

//A middleware to check user for protected routes.
const isAuthenticated = (req, res, next) => {
  if (req.isAuthenticated()) {
    return next();
  }
  res.redirect('/auth/easyauth/login');
};

passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((user, done) => done(null, user));

app.get('/', (req, res) => {
  res.send('EasyAuth Passport Home Page');
});

//Login Route
app.get(
  '/auth/easyauth/login',
  passport.authenticate('easyauth', {scope: 'openid'})
);

//Callback Route
app.get(
  '/auth/easyauth/callback',
  passport.authenticate('easyauth', {failureRedirect: '/failed'}),
  (req, res) => {
    res.redirect('/protectedroute');
  }
);

//Logout Route
app.get('/auth/easyauth/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      console.log(err);
    } else {
      res.send('Successfully Logged out.');
    }
  });
});

//Failed Route
app.get('/failed', (req, res) => {
  res.send('Login Failed');
});

//Protected success Route
app.get('/protectedroute', isAuthenticated, (req, res) => {
  res.send(`User: ${JSON.stringify(req.user)}`);
});

//Get EasyAuth profile
app.get('/easyauthprofile', isAuthenticated, async (req, res) => {
  try {
    const accessToken = req.user.tokenset.access_token;
    const response = await fetch(
      new URL('/tenantbackend/api/profile', process.env.EASYAUTH_DISCOVERY_URL),
      {
        method: 'GET',
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
      }
    );
    if (response.ok) {
      const userInfo = await response.json();
      res.send(JSON.stringify(userInfo));
    } else {
      res.send('Failed to fetch User Info.').status(response.status);
    }
  } catch (error) {
    res.send('Failed to fetch User Info').status(500);
  }
});

app.listen(PORT, () => {
  console.log(`Server started on http://127.0.0.1:${PORT}.`);
});
```

## Test the App

Run the application on `127.0.0.1`. Then open the URL [http://127.0.0.1:3000/auth/easyauth/login](http://127.0.0.1:3000/auth/easyauth/login){target=\_blank} to test the authentication.

=== "npm"

```bash
node index.js
```
