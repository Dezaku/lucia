---
title: "OAuth helpers (experimental)"
description: "Learn how to use the built-in OAuth helpers"
---

The OAuth integration provides helpers for implementing OAuth 2.0 authentication flow, including Open ID Connect. This page uses Github OAuth as an example but make sure to follow the provider's documentation.

A lot of APIs are still experimental and subject to breaking changes.

## Create authorization URL

You can create a new authorization url with a state with [`createOAuth2AuthorizationUrl()`](/reference/oauth/modules/main#createoauth2authorizationurl). This take the base authorization url, and returns the full url as the first item and an OAuth state as the second. If your provider doesn't support `state` parameter, it still should be fine to use this helper.

The state should be stored as a http-only cookie.

```ts
import { __experimental_createAuthorizationUrl } from "@lucia-auth/oauth";

// get url to redirect the user to, with the state
const [url, state] = await __experimental_createAuthorizationUrl(
	"https://github.com/login/oauth/authorize",
	{
		clientId,
		scope: ["user:email"], // empty array if none
		redirectUri
	}
);

setCookie("github_oauth_state", state, {
	path: "/",
	httpOnly: true, // only readable in the server
	maxAge: 60 * 60 // a reasonable expiration date
});

// redirect to authorization url
redirect(url);
```

### With PKCE code challenge

If your provider requires you to generate a PKCE code challenge, you can use [`createOAuth2AuthorizationUrlWithPKCE()`](/reference/oauth/modules/main#createoauth2authorizationurlwithpkce) instead, which returns the code verifier as the third item. This helper currently only supports SHA-256 as the code challenge method.

The code verifier should also be stored in a http-only cookie.

```ts
import { __experimental_createOAuth2AuthorizationUrlWithPKCE } from "@lucia-auth/oauth";

const [url, state] = await __experimental_createOAuth2AuthorizationUrlWithPKCE(
	url,
	{
		clientId,
		scope,
		redirectUri,
		codeChallengeMethod: "S256"
	}
);
```

### Additional configuration

You can provide your own state, and the `searchParams` option allows you to set any arbitrary query strings. Same options are also available for `createOAuth2AuthorizationUrl()`

```ts
await createOAuth2AuthorizationUrl("https://appleid.apple.com/auth/authorize", {
	clientId,
	scope,
	redirectUri,
	searchParams: {
		response_mode: "query" // set /?response_mode=query
	}
});
```

## Validate authorization code

Upon authentication, the provider will redirect the user back to your application. Make sure the state exists in the url query string and that it matches the one stored as cookies.

```ts
const state = requestUrl.searchParams.get("state");

// get state cookie we set when we got the authorization url
const stateCookie = getCookie("github_oauth_state");

// validate state
if (!state || !storedState || state !== storedState) throw new Error(); // invalid state
```

Extract the authorization code from the query string and verify it using [`validateOAuth2AuthorizationCode()`](/reference/oauth/modules/main#validateoauth2authorizationcode). This sends a request to the provided url and returns the JSON-parsed response body, which includes the access token. You can define the return type by passing a generic. This will throw a [`OAuthRequestError`](/reference/oauth/interfaces#oauthrequesterror) if the request fails.

```ts
import { __experimental_validateOAuth2AuthorizationCode } from "@lucia-auth/oauth";

type AuthorizationResult = {
	access_token: string;
};

const tokens =
	await __experimental_validateOAuth2AuthorizationCode<AuthorizationResult>(
		code,
		"https://github.com/login/oauth/access_token",
		{
			clientId,
			redirectUri // optional
		}
	);
const accessToken = tokens.access_token;
```

### Client password

If your provider takes a client password, there are 2 ways to verify the code. You can either sending the client secret in the body, or using the HTTP basic authentication scheme. This depends on the provider.

#### Send client secret in the body

Set `clientPassword.authenticateWith` to `"client_secret"` to send the client secret in the request body.

```ts
const tokens = await validateOAuth2AuthorizationCode<Result>(code, url, {
	clientId,
	clientPassword: {
		clientSecret,
		authenticateWith: "client_secret"
	}
});
```

#### Use HTTP basic authentication

You can send the base64 encoded client id and secret by setting `clientPassword.authenticateWith` to `"http_basic_auth"`/

```ts
const tokens = await validateOAuth2AuthorizationCode<Result>(code, url, {
	clientId,
	clientPassword: {
		clientSecret,
		authenticateWith: "http_basic_auth"
	}
});
```

### Send code verifier

Optionally, your provider may expect a code verifier instead or alongside the secret, in which case you can pass the code verifier stored as a cookie to `code_verifier`.

```ts
const codeVerifierCookie = getCookie("code_verifier");

const tokens = await validateOAuth2AuthorizationCode<Result>(code, url, {
	clientId,
	redirectUri,
	codeVerifier: codeVerifierCookie ?? ""
});
```

## Decode ID Token

If you're using Open ID Connect, you can decode the ID Token with [`decodeIdToken()`](/reference/oauth/modules/main#decodeidtoken). **This does NOT validate the ID token**. though decoding should be enough. This also takes a generic for the ID Token claims.

```ts
import { __experimental_decodeIdToken } from "@lucia-auth/oauth";

type Claims = {
	email: string;
};

const user = __experimental_decodeIdToken<Claims>(idToken);
const email = user.email;
```

## Initialize `ProviderUserAuth`

[`providerUserAuth()`](/reference/oauth/modules/main#provideruserauth) can be used to create a new [`ProviderUserAuth`](/reference/oauth/interfaces#provideruserauth) instance, which provides a few helpful utilities. It takes your Lucia `Auth` instance, the provider id (e.g. `"github"`), and the provider user id (e.g. Github user id).

```ts
import { auth } from "./lucia.js";
import { providerUserAuth } from "@lucia-auth/oauth";

const githubUserAuth = await providerUserAuth(auth, "github", githubUserId);
```

### Authenticate user with Lucia

[`ProviderUserAuth.existingUser`](/reference/oauth/interfaces#provideruserauth) will be defined if a Lucia user already exists for the authenticated provider account. If not, you can create a new Lucia user linked to the provider with [`ProviderUserAuth.createUser()`](/reference/oauth/interfaces#createuser). You can get the provider user data with `githubUser` for Github, etc.

```ts
const { existingUser, createUser, githubUser } =
	await githubAuth.validateCallback(code);

const getUser = async () => {
	if (existingUser) return existingUser;
	// create a new user if the user does not exist
	return await createUser({
		attributes: {
			username: githubUser.login
		}
	});
};

const user = await getUser();

// login user
const session = await auth.createSession({
	userId: user.userId,
	attributes: {}
});
```

### Add provider to existing user

Alternatively, you may want to add a new authentication method to an existing user by creating a new key. Calling [`ProviderUserAuth.createKey()`](/reference/oauth/interfaces#createkey) will create a new key linked to the provided user id.

```ts
const { existingUser, createKey } = await githubAuth.validateCallback(code);

if (!existingUser) {
	await createKey(currentUser.userId);
}
```

See [OAuth account linking](/guidebook/oauth-account-linking) guide for details.
