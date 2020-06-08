# Use the Humtum Platform to build Sentinel
## Start humtum-platform locally
To start:

* Clone the humtum source code
* Make sure you have rvm and yarn (Node.js) installed.
* Run rvm to install the ruby environment
```bash
rvm install 2.4.2
```
* After the rvm is installed, reenter the root project directory and run
```bash
bundle install
```
* Then goto the `./client` folder, then run
```bash
yarn install
```
* Go back to the root project directory, create `.env` file with the following content
```bash

export AUTH0_RUBY_CLIENT_ID=<Management API client ID>
export AUTH0_RUBY_CLIENT_SECRET=<Management API client Secret>>
```

* Run
```bash
source .env
```
* Run this to start the server
```bash

Rake db:migrate
Rake start
```
Now your humtum server should be running on localhost:3000

## Use humtum to create Sentinel backend



After creating the app, remember your app name and app id.

<img width="560" src="./appid.png"/>

## Create the Sentinel Chrome extension

Step 1. Clone [Humtum Chrome Extension Example](https://github.com/humtum-platform/HumtumChromeExtensionExample). The Sentinel Chrome Extension is in the `sentinel` folder. Make sure that the Extension ID stay constant using the following steps:

* In the `chrome://extensions/` page, pack the extension using `Pack extension` features by specifying the root location of the Chrome extension (`<YOUR-PATH>/HumtumChromeExtensionExample/sentinel`)
* Use [this webapp](https://robwu.nl/crxviewer/) to view the crx file
* Open the browser console to get following line

```
"key": "<your-chrome-extension-key>",
```
* copy this `"key": ...` into your `manifest.json`
* Load the unpacked extension and get your extension ID. This is the constant extension id

Step 2. Remember your extension ID for OIDC client registration later

Step 3. Create env.js file like [this](https://github.com/humtum-platform/HumtumChromeExtensionExample/blob/master/sentinel/src/background/env.js) with your Auth0 Credential.

Step 4. Load the unpacked extension into Chrome.

## Create the Sentinel electron app
* Read the [overview](https://auth0.com/blog/securing-electron-applications-with-openid-connect-and-oauth-2/#Electron-Overview) about Electron.
* The source code for this example is [here](https://github.com/humtum-platform/HumtumNodeExample/tree/master/sentinel)

### Register Sentinel client application
* First, you need to register a client ID using humtum-platform OIDC dynamic client registration endpoint, for example
```bash
curl --location --request POST 'localhost:3000/clients' \
--header 'Content-Type: application/json' \
--data-raw '{
	"client_name": "Sentinel",
	"redirect_uris": ["https://com.example.sendtinel", "chrome-extension://<your-extension-id>", "https://<extension-id>.chromiumapp.org/auth0"]
}'
```
* The result will be like. Save your credential in a safe place.
```json
{
    "tenant": "humtum",
    "global": false,
    "is_token_endpoint_ip_header_trusted": false,
    "is_first_party": false,
    "oidc_conformant": true,
    "callbacks":  ["https://com.example.sendtinel", "chrome-extension://<your-extension-id>", "https://<extension-id>.chromiumapp.org/auth0"],
    "refresh_token": {
        "token_lifetime": 2592000,
        "leeway": 3,
        "rotation_type": "rotating",
        "expiration_type": "expiring"
    },
    "name": "Sentinel",
    "sso_disabled": false,
    "cross_origin_auth": false,
    "encrypted": true,
    "signing_keys": "<keys>",
    "client_id": "<Client ID>",
    "callback_url_template": false,
    "client_secret": "<Client Secret>",
    "jwt_configuration": {
        "lifetime_in_seconds": 36000,
        "secret_encoded": false
    },
    "grant_types": ["authorization_code", "implicit", "refresh_token", "client_credentials"],
    "custom_login_page_on": true
}
```
### Implementing the Electron Client

Note: Our Sentinel example is a modification of [this blog](https://auth0.com/blog/securing-electron-applications-with-openid-connect-and-oauth-2/#Creating-the-Electron-Application). Read the blog before continue with this tutorial.

* Clone [this repo](https://github.com/humtum-platform/HumtumNodeExample)
* Go to `sentinel` folder. Run `npm i` to install all the dependencies.
* Create `env-variables.json` file for your application with the following
```json
{
  "apiIdentifier": "com.humtum.api.<YOUR-APP-NAME>",
  "auth0Domain": "humtum.auth0.com",
  "clientId": "<YOUR-CLIENT-ID>",
  "appId": "<YOUR-APP-ID>",
  "redirectUri": "<one-of-YOUR-redirect-uris>"
}
```
Note: Our app uses `keytar` to remember sensitive informations like the *refresh token* similar to the blog examples.

#### Authenticating Users in Electron Applications
Our example uses the [authorization code flow with pkce](https://auth0.com/docs/api-auth/tutorials/authorization-code-grant-pkce) and refresh token rotation enabled. In fact, all the clients that register through Humtum Platform has Refresh Token rotation enabled.

For this authorization flow, we need to implement in `./services/auth-service.js` the following functions:

* `getPKCEURLandSecret` : returns the complete URL of the Authorization Server that users have to visit to authenticate and the PKCE secret for the PKCE flow.
* `extractCode`: extract the code and the secret from callback URL returned by the Authorization Server
* `exchangeCodeForToken`: exchange the code from the Authorization Server for the access token and the refresh token. The refresh token needs to be stored securely on the device using keytar
* `refreshTokens`: use the refresh tokens to get the new access token


Code in `./main/auth-process.js` show how to authenticate using these functions

```javascript
// ./main/auth-process.js 
const {BrowserWindow} = require('electron');
const authService = require('../services/auth-service');
const {createAppWindow} = require('../main/app-process');
const humtum = require('../services/humtum')
const envVariables = require('../env-variables');


let win = null;

function createAuthWindow() {
  destroyAuthWin();

  // Create the browser window.
  win = new BrowserWindow({
    width: 1000,
    height: 600,
  });
}

function authenticateUsingAuthWindow() {
  const { apiIdentifier, redirectUri } = envVariables
  const {
    url,
    secret
  } = authService.getPKCEURLandSecret({
    audience: apiIdentifier,
    scope: "openid email profile offline_access read:appdata write:appdata",
    redirect_uri: redirectUri
  })

  win.loadURL(url);

  const {session: {webRequest}} = win.webContents;

  const filter = {
    urls: [
      `${redirectUri}/*`
    ]
  };

  webRequest.onBeforeRequest(filter, async ({url}) => {
    try {
      const {code , state} = authService.extractCode(url)
      await authService.exchangeCodeForToken(code, secret, state)
      // await authService.loadTokens(url);

      await humtum.enrollInApp(envVariables.appId, (e) => {
        createAppWindow();
        destroyAuthWin();
      }).then(data => {

        if (data == null) return
        authenticateUsingAuthWindow();

      })
    } catch(error){
      console.error(error);

    };

  });

  win.on('authenticated', () => {

    destroyAuthWin();
  });

  win.on('closed', () => {
    win = null;
  });
}

function destroyAuthWin() {
  if (!win) return;
  win.close();
  win = null;
}

function createLogoutWindow() {
  return new Promise(resolve => {
    const logoutWindow = new BrowserWindow({
      show: false,
    });

    logoutWindow.loadURL(authService.getLogOutUrl());

    logoutWindow.on('ready-to-show', async () => {
      logoutWindow.close();
      await authService.logout();
      resolve();
    });
  });
}

module.exports = {
  createAuthWindow,
  createLogoutWindow,
  authenticateUsingAuthWindow
};
```

First, use function `getPKCEURLandSecret` to get the authentication url and secret. The options passed to the function needs to contain the following fields.

```javascript
{
    audience: apiIdentifier,
    scope: "openid email profile offline_access read:appdata write:appdata",
    redirect_uri: redirectUri
}
```

Then, load the URL using the Electron window. We need to set up a filter so that the browser window can call a callback function when the Authorization Server call the callback url with the code and secret when the user authenticates successfully. This callback function should `extractCode` from the callback url and the `exchangeCodeForToken` so that our authentication service can get the access token. Once this is successful, you can use the humtum.js function to call the Humtum-platform API. In this example, we check whether the user has enrolled into our app. If she has, the app window can be launch. Otherwise, enroll the user and then reauthenticate to get the user's permission to access her app's data.

Once the user is authenticated and enrolled, we can start building Sentinel Electron Client!

### Implementing Sentinel Electron application.

Sentinel is an application that allows an expert to send a message about security issues for his / her followers. Therefore, the MVP for our app would have the following functions:

* Followers and Followig list
* Search people and Follow people
* Notifications for followers' request
* Approve and reject follow's request
* Broadcast a message
* Show user's profile

All these functions are implemented in the `./renderers` folder. To be more specifically,

* use `humtum.getFollowers` and `humtum.getFollowing` to get the friend list
* use `humtum.followOther` to send a follow request
* use `humtum.getFollowRequests` to get follow requests
* use `humtum.approveFollowRequest` and `humtum.rejectFollowRequest` to approve and reject follow request
* use `humtum.createMessage` to broadcast a message for our followers
* use `humtum.getSelf` to get the user profile
* use `humtum.searchUsersInApp` to search for users

### Implementing Sentinel Chrome Extension.

Our Chrome extension is used by the followers to get messages from expert which will be shown upon the launch of a website that is specified by the message. The code for the extension is at [this repo](https://github.com/humtum-platform/HumtumChromeExtensionExample/blob/master/sentinel). This code is also compatible with Firefox; however, in order to create a firefox extension, the correct redirect uri needs to be obtain from the Firefox in order to register it with the humtum-platform OIDC dynamic client registration endpoint.

Our Chrome extension has two components: `background` and `browser actions`. The `browser actions` helps the user login with humtum-platform. THe `background` process contains all the logic to login the user and to process all the messages received by the user.

#### Authentication with Auth0

Our Chrome extension uses a fork of [auth0-chrome](https://github.com/humtum-platform/auth0-chrome) library to authenticate.
First you need to create a env.js in the `src/background` folder.

```js
window.env = {
  AUTH0_DOMAIN: 'humtum.auth0.com',
  AUTH0_CLIENT_ID: '<your-client-id>',
};
```

and when the user press the login button, the `background` component receive an `authenticate` message and the below code run to authenticate with Auth0 and set up humtum to use the authentication token:

```javascript

// ./src/background/main.js
browser.runtime.onMessage.addListener(function (event) {
  if (event.type === 'authenticate') {

    // scope
    //  - openid if you want an id_token returned
    //  - offline_access if you want a refresh_token returned
    // device
    //  - required if requesting the offline_access scope.
    let options = {
      scope: "openid email profile offline_access read:appdata write:appdata",
      audience: "com.humtum.api.sentinel",
    };

    authzero = new Auth0Chrome(env.AUTH0_DOMAIN, env.AUTH0_CLIENT_ID)
    authzero
      .authenticate(options)
      .then(async (authResult) => {
          humtum.getAuth().storeAuthResult(authResult)
          humtum.getAuth().scheduleRenewal(refreshthetoken)
          let data = await humtum.getSelf()
          data && browser.notifications.create({
            type: 'basic',
            iconUrl: '/icons/icon128.png',
            title: 'Login Successful',
            message: `You can use the app now`
          });

          listenForMessage()
        }

      ).catch(function (err) {
        browser.notifications.create({
          type: 'basic',
          title: 'Login Failed',
          message: err.message,
          iconUrl: '/icons/icon128.png'
        });
      });

  } else if (event.type == "logout") {
    humtum.getCable().disconnect()
    filterWeb.clear()
    localStorage.webFilter = JSON.stringify(filterWeb.toJSON())

  }
});
```

Users login using the login button on the launch of the extension's browser action, and then they are set up to receive messages from experts. All of the logic for the Chrome extension to interact with humtum is in `src/background/main.js`

#### Implementing Sentinel

##### Receving messages and displaying the messages

We use `humtum.subscribeToChannel("MessagesChannel", ..., ... ,...)` to process the incoming messages. If the message has the correct format of:
```js
{
  website: string,
  websitemsg: string
}
```
The website and the message will be added to a lru storage. And then, using the [onUpdated](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/tabs/onUpdated) callback, for each time user visits a website, we can check the url against the stored message. If the url passes the check, the message will be displayed to the users.

##### Browser action

the `browser action` will display the profile if the user is logged in. Otherwise it would display the login button. [browser.runtime.sendMessage](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/runtime/sendMessage) is used to communicate with the browser backend.

```js

// ./src/browser_action/browser_action.js

function logout() {
  // Remove the idToken from storage
  localStorage.clear();
  browser.runtime.sendMessage({
    type: "logout"
  });
  main();
}

// Minimal jQuery
const $$ = document.querySelectorAll.bind(document);
const $ = document.querySelector.bind(document);


function renderProfileView(authResult) {
  $('.default').classList.add('hidden');
  $('.loading').classList.remove('hidden');
  fetch(`https://${env.AUTH0_DOMAIN}/userinfo`, {
    headers: {
      'Authorization': `Bearer ${authResult.access_token}`
    }
  }).then(resp => resp.json()).then((profile) => {
    ['picture', 'name', 'nickname'].forEach((key) => {

      const element = $('.' + key);
      if (element.nodeName === 'DIV') {
        element.style.backgroundImage = 'url(' + profile[key] + ')';
        return;
      }

      element.textContent = profile[key];
    });
    $('.loading').classList.add('hidden');
    $('.profile').classList.remove('hidden');
    $('.logout-button').addEventListener('click', logout);
  }).catch(logout);
}


function renderDefaultView() {
  $('.default').classList.remove('hidden');
  $('.profile').classList.add('hidden');
  $('.loading').classList.add('hidden');

  $('.login-button').addEventListener('click', () => {
    $('.default').classList.add('hidden');
    $('.loading').classList.remove('hidden');
    browser.runtime.sendMessage({
      type: "authenticate"
    });
  });
}

function main() {
  const authResult = JSON.parse(localStorage.humtumAuth || '{}');
  if (authResult.access_token && authResult.expires_in * 1000 - (Date.now() - authResult.start) > 0) {
    renderProfileView(authResult);
  } else {
    renderDefaultView();
  }
}


document.addEventListener('DOMContentLoaded', main);
```

##### Misc
In the case that the user has authenticate before, we automatically log the user in.
```js

// ./src/background.main.js
browser.runtime.onStartup.addListener(function () {
  if (humtum.getAuth().isAuthenticated() && humtum.getAuth().isValid()) {
    refreshthetoken()
    listenForMessage()
  }

  if (localStorage.webFilter) {
    JSON.parse(localStorage.webFilter).forEach(v => {
      filterWeb.set(v.key, v.value)
    })
  }
})

browser.runtime.onInstalled.addListener(function () {
  if (humtum.getAuth().isAuthenticated() && humtum.getAuth().isValid()) {
    refreshthetoken()
    listenForMessage()
  }

  if (localStorage.webFilter) {
    JSON.parse(localStorage.webFilter).forEach(v => {
      filterWeb.set(v.key, v.value)
    })
  }
})

```