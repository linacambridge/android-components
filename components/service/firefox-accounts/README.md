# [Android Components](../../../README.md) > Service > Firefox Accounts (FxA)

A library for integrating with Firefox Accounts.

## Motivation

The **Firefox Accounts Android Component** provides a way for Android applications to do the following:

* Obtain OAuth tokens that can be used to access the user's data in Mozilla-hosted services like Firefox Sync
* Fetch client-side scoped keys needed for end-to-end encryption of that data
* Fetch a user's profile to personalize the application

See also the [sample app](https://github.com/mozilla-mobile/android-components/tree/master/samples/firefox-accounts)
for help with integrating this component into your application.

## Usage

### Setting up the dependency

Use Gradle to download the library from [maven.mozilla.org](https://maven.mozilla.org/) ([Setup repository](../../../README.md#maven-repository)):

```Groovy
implementation "org.mozilla.components:service-firefox-accounts:{latest-version}"
```

### Start coding

> This tutorial is for version 0.15 of the FxA client.

First you need some OAuth information. Generate a `client_id`, `redirectUrl` and find out the scopes for your application.
See Firefox Account documentation for that. 

Once you have the OAuth info, you can start adding `FxAClient` to your Android project.
As part of the OAuth flow your application will be opening up a WebView or a Custom Tab.
Currently the SDK does not provide the WebView, you have to write it yourself.

Create a global `account` object: 

```kotlin
var account: FirefoxAccount? = null
```

You will need to save state for FxA in your app, this example just uses `SharedPreferences`. We suggest using the [Android Keystore]( https://developer.android.com/training/articles/keystore) for this data.
Define variables to help save state for FxA:

```kotlin
val STATE_PREFS_KEY = "fxaAppState"
val STATE_KEY = "fxaState"
```

Then you can write the following:

```kotlin
whenAccount = async {
    let savedState = getSharedPreferences(FXA_STATE_PREFS_KEY, Context.MODE_PRIVATE).getString(FXA_STATE_KEY, "")
    try {
        FirefoxAccount.fromJSONString(savedState).await()
    } catch (e: FxaException) {
        // Note: Config implements autoclosable, and must be disposed of.
        Config.custom(CONFIG_URL).await().use { config ->
            FirefoxAccount(config, CLIENT_ID, REDIRECT_URL)
        }
    }
}
```

The code above checks if you have some existing state for FxA, otherwise it configures it. It assigns `whenAccount`, which is a `Deferred` that resolves to the `FirefoxAccount` when it's complete.

You can now attempt to fetch the FxA profile. The first time the application starts it won't have any state, so
`account.getProfile()` will fail and proceed to the `account.beginOAuthFlow` branch and it will open the FxA OAuth login
in the web view.

```kotlin
launch {
    val account = whenAccount.await()
    try {
        val profile = account.getProfile().await()
        launch(UI) {
            txtView.text = profile.displayName
        }
    } catch (e: FxaException) {
        val url = account.beginOAuthFlow(scopes, wantsKeys)
        launch(UI) {
            openWebView(url)
        }
    }
}
```

When spawning the WebView, be sure to override the `OnPageStarted` function to intercept the redirect url and fetch the code + state parameters:

```kotlin
override fun onPageStarted(view: WebView?, url: String?, favicon: Bitmap?) {
    if (url != null && url.startsWith(redirectUrl)) {
        val uri = Uri.parse(url)
        val mCode = uri.getQueryParameter("code")
        val mState = uri.getQueryParameter("state")
        if (mCode != null && mState != null) {
            // Pass the code and state parameters back to your main activity
            listener?.onLoginComplete(mCode, mState, this@LoginFragment)
        }
    }

    super.onPageStarted(view, url, favicon)
}
```

Finally, complete the OAuth flow, try to retrieve the profile information, then save your login state once you've gotten valid profile information:

```kotlin
launch {
    val account = whenAccount.await()
    account.completeOAuthFlow(code, state).await()
    val profile = account.getProfile().await()
    launch(UI) {
        txtView.txt = profile.displayName
    }
    val json = account.toJSONString()
    getSharedPreferences(FXA_STATE_PREFS_KEY, Context.MODE_PRIVATE).edit()
        .putString(FXA_STATE_KEY, json).apply()
}
```

## License

    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/
