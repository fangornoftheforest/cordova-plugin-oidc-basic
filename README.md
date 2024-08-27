# Cordova Plugin OpenID Connect (OIDC) Basic

## Intro
Plug-in by mi-corporation, adapted for personal use. Might not fit your needs, but will be kept up to date.

## Dependencies
This plugin aims to be a thin wrapper around the following libraries/APIs:
- iOS -- [AppAuth-iOS](https://github.com/openid/AppAuth-iOS)
- Android -- [AppAuth-Android](https://github.com/openid/AppAuth-Android)

## Usage
```js
var req = {
    configuration: {
        authorizationEndpoint: "https://my-authorization-endpoint/authorize"
    },
    clientId: "my-client-id",
    scope: "openid",
    state: "my-state", // or leave absent and the plugin will generate random state by default
    redirectUrl: "my-redirect-url", // see notes on redirectUrl below
    responseType: "code" // authorization code flow
};
// Initiate authorization request
cordova.plugins.oidc.basic.presentAuthorizationRequest(req,
    function (resp) {
        // Success
        // Use the following info to perform your token request:
        // resp.authorizationCode -- The authorization code
        // resp.request.codeVerifier -- The codeVerifier. Generated by the plugin (currently no option to pass in or disable PKCE). Needed as part of the token request.
        // resp.request.nonce -- The nonce. Generated by the plugin (currently no option to pass in or disable). Needed as part of ID token validation.
    },
    function (err) {
        // Error. Inspect err.oidcType to see what kind. See below for more on errors and error types.
        // Example of handling a specific error type:
        if (err.oidcType === cordova.plugins.oidc.basic.ErrorType.USER_CANCELLED) {
            // User cancelled the authorizationr request
        }
        // Can handle other error types as needed
    }
);
```

## Setup
### Configuring the `redirectUrl`
To successfully receive redirects on all platforms, your `redirectUrl` must adhere to the following conventions:
- For iOS, use a custom URL scheme. No pre-registration of your custom scheme is needed provided you're targeting
  iOS 11+. For iOS 10 and below, you need to declare your custom scheme in your `*-Info.plist` file. You can use
  the [Custom-URL-scheme plugin](https://github.com/EddyVerbruggen/Custom-URL-scheme) to do this. See
  https://github.com/EddyVerbruggen/Custom-URL-scheme#2-installation.

- For Android, you can use a custom URL scheme or an [app link](https://developer.android.com/training/app-links).
  Either way, the redirect url must be configured at plugin add time via one of the variables
  `ANDROID_REDIRECT_SCHEME` or `ANDROID_REDIRECT_DATA_ELEMENT` (either one is sufficient).

  To use a custom URL scheme, e.g. `com.example.myapp`, use the `ANDROID_REDIRECT_SCHEME` variable, e.g.
    ```
    cordova plugin add https://github.com/mi-corporation/cordova-plugin-oidc-basic.git --variable ANDROID_REDIRECT_SCHEME=com.example.myapp
    ```
  To use an [app link](https://developer.android.com/training/app-links) for
  Android M and above, set the `ANDROID_REDIRECT_DATA_ELEMENT` variable to the matching XML `<data>`
  element. Make sure to escape this XML appropriately for whatever command shell you're using. E.g.
  if your app link is `https://myapp.example.com/oidc_redirect` and you're using zsh/bash/etc, do
    ```
    cordova plugin add https://github.com/mi-corporation/cordova-plugin-oidc-basic.git --variable ANDROID_REDIRECT_DATA_ELEMENT="<data android:scheme=\"https\" android:host=\"myapp.example.com\" android:path=\"/oidc_redirect\" />"
    ```
  Or on a Windows command prompt, do
    ```
    cordova plugin add https://github.com/mi-corporation/cordova-plugin-oidc-basic.git --variable ANDROID_REDIRECT_DATA_ELEMENT="<data android:scheme=""https"" android:host=""myapp.example.com"" android:path=""/oidc_redirect"" />"
    ```
  All `<data>` element attributes are supported. See https://developer.android.com/guide/topics/manifest/data-element
- For Windows, the `redirectUrl` can be any URL. No preregistration of your `redirectUrl`
  is required.

### Tips on CocoaPods Setup on iOS
The iOS plugin uses [CocoaPods](https://cocoapods.org/) to bundle [AppAuth-iOS](https://github.com/openid/AppAuth-iOS). If you haven't used CocoaPods before, you may find the following setup steps helpful:
- Download CocoaPods onto your iOS development machine either from https://cocoapods.org/app or by
  following the steps at https://guides.cocoapods.org/using/getting-started.html. The first link installs a small GUI application. When it prompts you to install the CLI tool, say yes. The second link installs just
  the CLI tool.
- You'll need to clone the CocoaPods Master repo before you perform iOS builds. Follow the steps at
  https://stackoverflow.com/a/39904450 to minimize the size of the clone. Be aware that "minimize" is relative:
  as of March 2020, a shallow clone still took 1.94GB on my box.
- After you `cordova plugin add` this plugin, it'll create a new `*.xcworkspace` file (e.g. `MyApp.xcworkspace`) next to your existing `*.xcodeproj` file in `.../platforms/ios`. Open the `*.xcworkspace` file
  in XCode instead of the `*.xcodeproj` file. If you open just the `*.xcodeproj` file, XCode will fail to
  include AppAuth-iOS when you try to build.

## Supported Features
- OpenID Connect authentication requests and OAuth 2.0 authorization requests.
  Use `presentAuthorizationRequest`.

  Authorization code flow, implicit flow, and OpenID Connect hybrid flow are all supported.
  See https://openid.net/specs/openid-connect-core-1_0.html#Authentication
  and https://tools.ietf.org/html/rfc6749#section-4.

  Supported on all platforms.

- OpenID Connect RP-initiated logout.
  Use `presentEndSessionRequest`.

  See https://openid.net/specs/openid-connect-session-1_0.html#RPLogout.

  Supported on iOS and Windows. *Not supported on Android* (due to missing AppAuth-Android support,
  see https://github.com/openid/AppAuth-Android/issues/374).

This plugin does not (and likely will not) provide APIs to perform a token request as part of the
authorization code or hybrid flows or validate the OpenID Connect ID token returned from a token request or
as part of the implicit flow. Since those steps require no user interaction, there's no reason they need
Cordova plugin support. If you're looking to perform those steps in app, consider resources such as
- [AppAuth-JS](https://github.com/openid/AppAuth-JS), which supports sending token requests
- [jwt-decode](https://github.com/auth0/jwt-decode), which supports decoding (though not validating) JWT
  tokens

Or you can perform your token request and ID token validation server-side. For a node server, consider
resources such as
- [AppAuth-JS](https://github.com/openid/AppAuth-JS), which works in Node as well as in browser
- [node-jsonwebtoken](https://github.com/auth0/node-jsonwebtoken), which supports decoding and validating JWT
  tokens



## Concurrent Requests
cordova-plugin-oidc-basic forbids concurrent calls to either `presentAuthorizationRequest` or
`presentEndSessionRequest`. If a call to `presentAuthorizationRequest` or `presentEndSessionRequest` is
already pending, then cordova-plugin-oidc-basic will reject a 2nd call to either method with an error of
type `UNSENDABLE_REQUEST`. This is partly a matter of implementation convenience: On some platforms (e.g.
older iOS), the plugin must manually route incoming callbacks from the OS to pending requests, and that routing is simpler if there can be at most one such request. It also prevents bombarding the user with
multiple concurrent login screens.

## Errors
### Error types
All error types are available via `cordova.plugins.oidc.basic.ErrorType`. Explanation of types:
- `cordova.plugins.oidc.basic.ErrorType.UNSENDABLE_REQUEST` -- The calling code did something wrong,
  e.g. passed an invalid authorization request, such that the request couldn't even be sent to the
  authorization server.
- `cordova.plugins.oidc.basic.ErrorType.ERROR_RESPONSE` -- The authorization server returned an error
  response as specified in https://tools.ietf.org/html/rfc6749#section-4.1.2.1.
- `cordova.plugins.oidc.basic.ErrorType.INVALID_RESPONSE` -- The authorization server returned an
  invalid response not in keeping w/ the OpenID Connect/OAuth 2.0 spec.
- `cordova.plugins.oidc.basic.ErrorType.HTTP_ERROR` -- There was an HTTP error completing the request.
- `cordova.plugins.oidc.basic.ErrorType.USER_CANCELLED` -- The user cancelled the request.
- `cordova.plugins.oidc.basic.ErrorType.UNEXPECTED_ERROR` -- There was an unexpected error completing
  the request.

### Error interface
All errors passed to the provided error callback are normal JS `Error`s extended with the following
properties:
- `oidcType` String. One of the `cordova.plugins.oidc.basic.ErrorType` values.
- `oidcDetails` String. Optional. Additional details about the error.
- `oidcResponse` Object. Optional. If applicable, the error response from the authorization server.

## API Documentation
See the [Typescript type definitions](types/index.d.ts).
