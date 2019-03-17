# What Is UserIn

__*UserIn*__ is an open source middleware REST API built in NodeJS with Express. __*UserIn*__ lets App engineers to implement custom login/register feature using Identity Providers (IdPs) such as Facebook, Google, Github and many others. __*UserIn*__ aims to be an open source alternative to Auth0, Firebase Authentication and AWS Cognito. It was initially designed to be hosted as a microservice (though its codebase could be integrated into any NodeJS system) in a serverless architecture (e.g., AWS Lambda, Google Functions, App Engine, ...).

The reason for this project is to offer alternatives to SaaS such as Auth0, Firebase Authentication, AWS Cognito which require by default to store App's users in their own data store. UserIn assumes that software engineers are comfortable with building CRUD REST APIs to store and manage users (e.g., GET users by ID, POST create new users) while securing those APIs with an API key. This design means that software engineers are responsibile to store user's data themselves while the boilerplate to exchange OAuth messages between all parties is abstracted behind UserIn. 

The manual steps left to the App engineers are:

* [1. Create an App In The IdP](#1-create-an-app-in-the-idp) - This has to be done for each IdP your app needs to support. This would have had to be done for Auth0, Firebase Authentication, AWS Cognito anyway.
* [2. Configuring The UserIn Middleware With The IdP Secrets](#2-configuring-the-userin-middleware-with-the-idp-secrets) - This is just a matter of updating the `userinrc.json` file with the secrets acquired in the previous step. This also would have had to be done for Auth0, Firebase Authentication, AWS Cognito anyway.
* [3. Generate a UserIn API Key To Communicate Safely With The App System](#3-Generate-a-UserIn-API-Key-To-Communicate-Safely-With-The-App-System) - This API key allows to communicate safely between your existing backend and UserIn.
* [4. Add a Few REST APIs In The App To Communicate With The UserIn Middleware](#4-add-a-few-rest-apis-in-the-app-to-communicate-with-the-userin-middleware) - Those REST APIs can be developped any ways the softwre engineers chose to, but they need to implement very specific signatures. 

# Getting Started
## 1. Clone this project

```
git clone https://github.com/nicolasdao/userin
cd userin
```

## 2. Get the App ID, App Secret, and configure the Redirect URIs for each IdP

Login to your IdP. We've detailed how to configure an new App for all the following IdPs:

* [Facebook](#facebook)
* [Google](#google)
* [LinkedIn](#linkedin)
* [GitHub](#github)

> IMPORTANT: Read carefully the note on how to set up the redirect URI.

## 3. Create a `userinrc.json` & Configure It

Add the `userinrc.json` in the root folder. Use the App ID and the App secret collected in the previous step. Here is an example:

```js
{
	"userPortal": {
		"api": "http://localhost:3500/user/in",
		"key": "EhZVzt1r9POWyV99Y3D3029k3tnkTApG6xInATpj"
	},
	"schemes": {
		"facebook": {
			"appId": 1234567891011121314,
			"appSecret": "abcdefghijklmnopqrstuvwxyz"
		},
		"google": {
			"appId": 987654321,
			"appSecret": "zfcwfceefeqfrceffre"
		}
	},
	"onSuccess": {
		"whatever": "you-want-in-case-of-successful-auth"
	},
	"onError": {
		"somethingelse": "in-case-failing-to-auth"
	}
}
```

Where:

| Property 				| Description |
|-----------------------|-------------|
| `userPortal.api` 		| HTTP POST endpoint. Expect to receive a `user` object. Creating this web endpoint is the App engineer's responsibility. More details about this in section |
| `userPortal.key`		| Optional, but highly recommended. This key allows to secure the communication between `userIn` and the `userPortal.api`. When specified, a header named `x-api-key` is passed during the HTTP POST. The App engineer should only allow POST requests if that header is set with the correct value, or return a 403. |
| `schemes.facebook` 	| This object represents the Identity Provider. It contains two properties: `appId` and `appSecret`. All IdPs follow the same schema. Currently supported IdPs: `facebook`, `google`, `linkedin` and `github`. |
| `onSuccess` 			| Optional. Custom object returned to the client each time a successfull login is achieved. |
| `onError` 			| Optional. Custom object returned to the client each time a failed login happens. |

## 4. Add a new web enpoint into your existing App

In this step, we suppose you have an existing web API powering your App. Everything works fine, except there is no user login and your web endpoints are not protected by a JWT token, meaning anybody can access them. Consider this simplistic example:

```js
const { app } = require('@neap/funky')

app.get('/sensitive/data', (req,res) => res.status(200).send('Special data for logged in users only'))

eval(app.listen(3500))
```

Run this code, and then use curl to query the web endpoint:

```
node index.js
```
```
curl http://localhost:3500/sensitive/data
```

As you can see, the `/sensitive/data` endpoint is not protected, and anybody can access the sensitive data.

Now, let's change the code as follow:

```js
const { app } = require('@neap/funky')
const Encryption = require('jwt-pwd')
const { apiKeyHandler, jwt } = new Encryption({ jwtSecret: '5NJqs6z4fMxvVK2IOTaePyGCPWvhL9MMX/o7nk2/9Ko/5jYuX+hUdfkmIzVAj6awtWk=' })

const _userStore = []

const _getUser = (id, authMethod) => _userStore.find(u => u && u.id == id && u.authMethod ==  authMethod)

app.post('/user/in', apiKeyHandler({ key:'x-api-key', value:'EhZVzt1r9POWyV99Y3D3029k3tnkTApG6xInATp' }), (req,res) => {
	const { user } = req.params || {}
	const { id, authMethod, email } = user || {}
	const u = _getUser(id, authMethod)
	if (!u)
		_userStore.push(user)
	
	jwt.create({ id, authMethod, email }).then(token => res.status(200).send({ token }))
})

app.get('/sensitive/data', (req,res) => res.status(200).send('Special data for logged in users only'))

eval(app.listen(3500))
```

Run:
```
node index.js
```
```
curl http://localhost:3500/sensitive/data
```

This time, you'll receive a 403 response with this error message: `Unauthorized access. Missing bearer token. Header 'x-token' not found.`

To access to this GET endpoint, you'll need to get a JWT token first. Run this curl method:

```
curl -d '{"id":"1", "firstName":"Nic", "email":"nic@neap.co", "authMethod":"default"}' -H "x-api-key: EhZVzt1r9POWyV99Y3D3029k3tnkTApG6xInATp" -X POST http://localhost:3500/user/in
```

Extract the token from the response, and then run:

```
curl -H "x-token: bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE1NTI4Mjg3NjB9.V56ZvG1VXmM2lrhm3NVFgmACxhylyLbgTTmFpQccF9k" http://localhost:3500/sensitive/data
```


# What App Engineers Must Still Do Manually

* [1. Create an App In The IdP](#1-create-an-app-in-the-idp) - This has to be done for each IdP your app wants to support.
* [2. Configuring The UserIn Middleware With The IdP Secrets](#2-configuring-the-userin-middleware-with-the-idp-secrets) - This is just a matter of updating the `userinrc.json` file with the secrets acquired in the previous step.
* [3. Generate a UserIn API Key To Communicate Safely With The App System](#3-Generate-a-UserIn-API-Key-To-Communicate-Safely-With-The-App-System) - This API key allows to communicate safely between your existing backend and UserIn.
* [4. Add a Few REST APIs In The App To Communicate With The UserIn Middleware](#4-add-a-few-rest-apis-in-the-app-to-communicate-with-the-userin-middleware) - Those REST APIs can be developped any ways the softwre engineers chose to, but they need to implement very specific signatures. 

## 1. Create an App In The IdP
### Facebook
#### Purpose 

* Acquire an __*App ID*__ and an __*App Secret*__.
* Configure __*Redirect URIs*__ (more info about redirect URIs under section [Concepts & Jargon](#concepts--jargon) / [Redirect URI](#redirect-uri)). 

#### Steps

1. Browse to [https://developers.facebook.com/](https://developers.facebook.com/).
2. In the top menu, expand __*My Apps*__, click on the __*Add New App*__ link and fill up the form. Once submitted, you'll be redirected to your new app dashboard page.
3. Optionally, create a test version of your new app (recommended) to ease testing during the development stage. In the top left corner, click on the app name to expand the menu. At the bottom of that menu, click on the __*Create Test App*__ button.
4. Get the __App ID__ and the __App Secret__
	1. The App ID is easily foundable at the top of each page.
	2. The App Secret is under __*Settings/Basic*__ section in the left menu. 
	3. Create or configure the `userinrc.json` file under the `userin` root folder as follow:

	```js
	{
		"schemes": {
			"facebook": {
				"appId": "your-app-id",
				"appSecret": "your-app-secret"
			}
		}
	}
	```

5. Add valid OAuth redirect URI :
	1. In the left menu, expand the __*Facebook Login*__ tab and click on __*Settings*__. 
	2. Under __*Valid OAuth Redirect URIs*__, enter [your-origin/facebook/oauth2callback](your-origin/facebook/oauth2callback), where `your-origin` depends on your hosting configuration. In development mode, _userIn_ is probably hosted on your local machine and the redirect URI probably looks like [http://localhost:3000/facebook/oauth2callback](http://localhost:3000/facebook/oauth2callback). When releasing your app in production, _userIn_ will most likely be hosted under your custom domain (e.g., youcool.com). You will have to change the redirect URI to [https://youcool.com/facebook/oauth2callback](https://youcool.com/facebook/oauth2callback).

> NOTE: Step 5 is not required in Development mode (this is the default when the App is created). This assumes that your app is hosted locally using _http://localhost_. When going live, step 5 is required.

### Google
#### Purpose 

* Acquire an __*App ID*__ and an __*App Secret*__.
* Configure a __*Consent Screen*__. That screen acts as a disclaimer to inform the user of the implication of using Google as an IdP to sign-in to your App.
* Configure __*Redirect URIs*__ (more info about redirect URIs under section [Concepts & Jargon](#concepts--jargon) / [Redirect URI](#redirect-uri)). 

#### Steps

1. Sign in to your Google Cloud console at [https://console.cloud.google.com](https://console.cloud.google.com).
2. Choose the project tied to your app, or create a new one.
3. Once your project is created/selected, expand the left menu and select __*APIs & Services / Credentials*__
4. In the _Credentials_ page, select the _Credentials_ tab, click on the __*Create credentials*__ button and select __*OAuth Client ID*__. Fill up the form:
	1. _Name_: This is not really important and will not be displayed to your user. You can leave the default.
	2. _Authorized JavaScript origins_: This is optional but highly recommended before going live.
	3. _Authorized redirect URIs_: This is required. Enter [your-origin/google/oauth2callback](your-origin/google/oauth2callback), where `your-origin` depends on your hosting configuration. In development mode, _userIn_ is probably hosted on your local machine and the redirect URI probably looks like [http://localhost:3000/google/oauth2callback](http://localhost:3000/google/oauth2callback). When releasing your app in production, _userIn_ will most likely be hosted under your custom domain (e.g., youcool.com). You will have to change the redirect URI to [https://youcool.com/google/oauth2callback](https://youcool.com/google/oauth2callback).
5. After completing the step above, a confirmation screen pops up. Copy the __*client ID*__ (i.e., the App ID) and the __*client secret*__ (i.e., the App Secret). Create or configure the `userinrc.json` file under the `userin` root folder as follow:

	```js
	{
		"schemes": {
			"google": {
				"appId": "your-app-id",
				"appSecret": "your-app-secret"
			}
		}
	}
	```
6. In the _Credentials_ page, select the _OAuth consent screen_ tab. Fill up the form depending on your requirements. Make sure you update the __*Application name*__ to your App name so that your App users see that name in the consent screen. You can also add your brand in the consent screen by uploading your App logo. Don't forget to click the __*Save*__ button at the bottom to apply your changes.

### LinkedIn
#### Purpose 

* Acquire an __*App ID*__ and an __*App Secret*__.
* Configure __*Redirect URIs*__ (more info about redirect URIs under section [Concepts & Jargon](#concepts--jargon) / [Redirect URI](#redirect-uri)). 

#### Steps

1. Sign in to your LinkedIn account and then browse to [https://www.linkedin.com/developers/apps](https://www.linkedin.com/developers/apps) to either create a new App or access any existing ones. For the sake of this tutorial, the next steps only focus on creating a new App.
2. In the top right corner of the __*My Apps*__ page, click on the __*Create app*__ button.
3. Fill up the form and then click the __*Create app*__ button at the bottom.
4. Once the the App is created, you are redirected to the App's page. In that page, select the __*Auth*__ tab and copy the __*Client ID*__ (i.e., the App ID) and the __*Client secret*__ (i.e., the App Secret). Create or configure the `userinrc.json` file under the `userin` root folder as follow:

	```js
	{
		"schemes": {
			"linkedin": {
				"appId": "your-app-id",
				"appSecret": "your-app-secret"
			}
		}
	}
	```
5. Still in the __*Auth*__ tab, under the __*OAuth 2.0 settings*__ section, enter the redirect URI [your-origin/linkedin/oauth2callback](your-origin/linkedin/oauth2callback), where `your-origin` depends on your hosting configuration. In development mode, _userIn_ is probably hosted on your local machine and the redirect URI probably looks like [http://localhost:3000/linkedin/oauth2callback](http://localhost:3000/linkedin/oauth2callback). When releasing your app in production, _userIn_ will most likely be hosted under your custom domain (e.g., youcool.com). You will have to change the redirect URI to [https://youcool.com/linkedin/oauth2callback](https://youcool.com/linkedin/oauth2callback).

### Github
#### Purpose 

* Acquire an __*App ID*__ and an __*App Secret*__.
* Configure __*Redirect URIs*__ (more info about redirect URIs under section [Concepts & Jargon](#concepts--jargon) / [Redirect URI](#redirect-uri)). 

#### Steps

1. Sign in to your Github account and then browse to [https://github.com/settings/apps](https://github.com/settings/apps) to either create a new App or access any existing ones. For the sake of this tutorial, the next steps only focus on creating a new App.
2. In the top right corner of the __*GitHub Apps*__ page, click on the __*New GitHub App*__ button.
3. Fill up the form and then click the __*Create GitHub App*__ button at the bottom. The most important field to fill is the __*User authorization callback URL*__. Enter the redirect URI [your-origin/github/oauth2callback](your-origin/github/oauth2callback), where `your-origin` depends on your hosting configuration. In development mode, _userIn_ is probably hosted on your local machine and the redirect URI probably looks like [http://localhost:3000/github/oauth2callback](http://localhost:3000/github/oauth2callback). When releasing your app in production, _userIn_ will most likely be hosted under your custom domain (e.g., youcool.com). You will have to change the redirect URI to [https://youcool.com/github/oauth2callback](https://youcool.com/github/oauth2callback).

> NOTE: The App creation form forces you to enter a _Homepage URL_ and a _Webhook URL_. If you don't have any, that's not important. Just enter random URIs (e.g., Homepage URL: [https://leavemealone.com](https://leavemealone.com) Webhook URL: [https://leavemealone.com](https://leavemealone.com))

4. Once the the App is created, you are redirected to the App's page. In that page, copy the __*Client ID*__ (i.e., the App ID) and the __*Client secret*__ (i.e., the App Secret). Create or configure the `userinrc.json` file under the `userin` root folder as follow:

	```js
	{
		"schemes": {
			"github": {
				"appId": "your-app-id",
				"appSecret": "your-app-secret"
			}
		}
	}
	```

## 2. Configuring The UserIn Middleware With The IdP Secrets

## 3. Generate a UserIn API Key To Communicate Safely With The App System

## 4. Add a Few REST APIs In The App To Communicate With The UserIn Middleware

# Theory & Concepts
## The Basics
### Identity Provider

In the broad sense of the term, an identity provider (IdP) is an entity that can issue a referral stating that a user is indeed the owner of an identity piece (e.g., email address). The most well known IdPs are Facebook, Google and Github.

### OAuth is an _Authorization Protocol_ NOT an _Authentication Protocol_

Contrary to [OpenID](https://en.wikipedia.org/wiki/OpenID) which is a pure authentication protocol, OAuth's main purpose is to authorize 3rd parties to be granted limited access to APIs. When an App offers its user to register using his/her Facebook or Google account, that does not mean the user proves his/her identity through Facebook or Google. Instead, that means that the user grants that App limited access to his/her Facebook or Google account. However, as of today (2019), gaining limiting access to an Identity Provider's API is considered _good enough_ to prove one's identity. That's what is referred to as a __*pseudo authentication*__ method. This project aims at facilitating the implementation of such _pseudo authentication_ method.

### Using OAuth With an IdP Is Not An App Login Full Picture

When OAuth with an IdP is involved in an App login workflow, the entire workflow is usually made of two steps:
1. __*Pseudo Authentication*__: This is the OAuth bit referred by many articles online. This step is about using an IdP such as Facebook to get the user details faster, skipping the password step, or simply getting access to the IdP API.
2. __*App Access Authentication & Authorization*__: This has nothing to do with OAuth, though OAuth could be used to perform that task if the App engineers choose to. The App engineers is usually required to build a security mechanism that will identify the user and grant the right level of access to resources. 

Many tools on the market (Auth0, Firebase Authentication, AWS Cognito, ...) help making the _pseudo authentication_ step easier, but in any case, the App engineers are still responsible to code the _App Access Authentication & Authorization_ step. On a high level, the _App Access Authentication & Authorization_ usually consists in, but is not limited to:
* Authenticating the user request (e.g., verifying a valid token is passed with each request).
* Authorizing the request (e.g., validating that the user associated with the token is authorized to access the resource).

			// NOT SURE ABOUT BELOW
			Contrary to what a lot of people believe, OAuth is not used to login users to Apps. Instead, it represents a shortcut to capture user's details so that THE APP CAN GENERATE ITS OWN AUTH TOKEN. Yes, you read correctly. OAuth does not magically absolve the app to perform any authentication and authorization processing. The app still has to:
			1. Generate its own token (using any method the App engineers want).
			2. Validate that token for each request.

			Those two steps have nothing to do with OAuth, and no secured apps are free from implementing them in one way or the other. So why do we need OAuth then? There are two main reasons:
			1. Users usually hate to create a new username and memorize passwords. Instead of asking them to do so, we can ask an IdP the user is already connected to to send details about him/her and use those details to create a new account or let them access the app without entering a password. 
			2. The app may need to access certain IdP's APIs. In which case, the app will need an OAuth token issued by the IdP.

## Workflow To Use OAuth To Help Users To Login 

Regardless of the IdP (e.g., Facebook. Google, Github), the _pseudo authentication_ workflow is the same.

1. App user clicks on _Login with <IdP>_ in the App login page, which directs the user to the IdP OAuth login page. That request to the IdP OAuth page passes multiple parameters including but not limited to:
	1. App ID and App Secret used for security purposes.
	2. Redirect URI to inform the IdP about the location where the user must be redirected after the IdP authorization validation.

> NOTE: If the user already logged in with the IdP in the past, that step might be so quick that it may not be visible. On the other hand, if that's the first time the user uses the IdP, a consent screen prompts to accept to grant the App access to the IdP.

2. After successfully verifying the user, the IdP redirects the user to the _redirect URI_ that was passed in the previous step. The IdP usually appends a query string to that redirect URI containing an access token. This access token allows the user to make another secured HTTP POST request to obtain IdP's data:
		1. An IdP token (OAuth token) that can be used to access the IdP's API.
		2. A token expiry date that defines when the OAuth token expires.
		3. A renewal token that can be used to renew the Oauth token after it has expired.
		4. Claims regarding the current IdP's user (e.g., email, first name, last name, ...). The type of claims depend on the type of access the App is requesting in the first place.




			// NOT SURE ABOUT BELOW
			1. Use OAuth to delegate identity verification to an IdP. This identity is relative to the IdP. That means that OAuth should return an IdP ID which uniquely identify that user for that specific IdP (e.g., Facebook ID, Google ID, ...). This ID can then be stored in your own user database when the user is created for the first time. For a user using Facebook as an IdP, next time that user logs in to the app using Facebook, that Facebook ID will be used to retrieve the app user's details.
			2. Create a new security token based on the App's user details. This step is post OAuth and has nothing to do with OAuth. At this stage, either the user is new and details have been collected, or the user is an existing one and his details have been retrieved from the App's database using the IdP ID. Using those credentials, the App is responsible to create a new token using any method. 

			Authenticating and authorizing users to an App require the 2 steps above. The first one uses OAuth, and the second one uses an arbitrary method based on what the App engineers want to accomplish. 

			This project aims at automating the first OAuth step, and facilitating the second.

			Regardless of the IdP (e.g., Facebook. Google, Github), the _pseudo authentication_ workflow is the same:
			1. Click _login with <IdP>_
				1. This step redirect the App to the IdP specific page for deal with authorization requests.
				2. If the user is already logged in to the IdP, the IdP redirects to the App's redirect URL (more about this later). If the user is not logged in, then he/she must log in. Upon successful login, the IdP redirects to the App's redirect URL. In both cases, the IdP passes mutliple pieces of data to the App's redirect URL:
					1. An IdP token (OAuth token) that can be used to access the IdP's API.
					2. A token expiry date that defines when the OAuth token expires.
					3. A renewal token that can be used to renew the Oauth token after it has expired.
					4. Claims regarding the current IdP's user (e.g., email, first name, last name, ...). The type of claims depend on the type of access the App is requesting in the first place. More details about this in section 

			2. Receive IdP authorization confirmation with user's data.
			3. Create a new account or let the user in:
				1. Now the user's data have been collected,


# How To
## How To Add a Authentication Method?

This can be done by adding one or many of the following supported scheme in the `userinrc.json`:
- __*default*__
- __*facebook*__
- __*google*__
- __*github*__

where _default_ represent the standard username/password authentication method.

Configuring an authentication portal that supports both Facebook and the default username/password would require a `userinrc.json` file similar to the following:

```js
{
	"schemes": [
		"default",
		"facebook"
	]
}
```

## How To Generate API Key?
__*UserIn__ ships with a utility that generates API keys. In your terminal, browse to the _UserIn_ root folder, and run the following command:

```
npm run key
```

> You're free to generate your API using any method you want. The command above is provided to users for convenience.

# Concepts & Jargon
## Redirect URI

A _Redirect URI_ is a URI that receives the IdP's response with the user details after successful authorizations. That URI is configured in your code. Because, in theory, any URI can be configured, the IdP wants to guarentee the App engineer that they can control the permitted Redirect URI. Therefore, most IdPs require the App engineers to whitelist their redirect URIs durin the App creation step.