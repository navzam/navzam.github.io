---
layout: post
title: "AAD login page in iframes"
date: 2021-03-12
---

If you try loading the Azure Active Directory (AAD) login page inside an iframe, you'll likely encounter errors due to defensive measures taken by AAD to prevent [clickjacking attacks](https://owasp.org/www-community/attacks/Clickjacking). In short, a malicious site could load the login page in a transparent iframe, overlay it on top of some dummy UI elements, and trick users into granting it access to the user's data.

<!--more-->

But it isn't as simple as saying "the AAD login page can never be iframed." There are cases where it's possible to conduct an auth flow through `https://login.microsoftonline.com` in an iframe. Even Microsoft's own auth library for JavaScript (MSAL.js) uses hidden iframes to achieve silent token acquisition. This got me wondering... What can and can't you do with the AAD login page in an iframe?

# Setup

I start by registering an AAD app through the Azure portal. I won't go through the whole process here, but you can check out the steps in the [AAD documentation for registering an application](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app). I'm running everything locally, so I set the redirect URI to `http://localhost:5500/auth-end.html`.

To make sure everything is registered properly, I craft a login URL for an [auth code flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow) that I can use to test a normal (non-iframed) flow:


```
https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
client_id=ec33faa3-71b7-4675-a053-0ddfa673e40d
&response_type=code
&redirect_uri=http://localhost:5500/auth-end.html
&response_mode=query
&scope=openid%20offline_access%20https%3A%2F%2Fgraph.microsoft.com%2Fuser.read
&state=12345
```

Putting this into my browser directly, I'm taken through the AAD login process and then redirected to my `auth-end` page with an auth code, as expected.

# Trying the flow in an iframe

What happens if we try to conduct the same flow in an `iframe`? To test this, I created an HTML page (`index.html`) containing an `iframe` that points to the same login URL above:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AAD iframe test app</title>
</head>
<body>
    <h2>AAD iframe test</h2>
    <iframe
        src="https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=ec33faa3-71b7-4675-a053-0ddfa673e40d&response_type=code&redirect_uri=http://localhost:5500/auth-end.html&response_mode=query&scope=openid%20offline_access%20https%3A%2F%2Fgraph.microsoft.com%2Fuser.read"
        width="100%"
        height="300px"
        style="border:1px solid black"
    >
    </iframe>
</body>
</html>
```

I also created a second page (`auth-end.html`) for the auth flow to redirect to. It displays the value of the `code` or `error` query string parameter, which makes it easier to see if the flow succeeded or failed:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <p id="pElement"></p>
    <script>
        let urlParams = new URLSearchParams(window.location.search);
        let code = urlParams.get('code');
        if (code !== null) {
            document.getElementById("pElement").textContent = `Successfully got auth code: ${code.substr(0, 10)}...`;
        } else {
            let error = urlParams.get('error');
            document.getElementById("pElement").textContent = `Failed to get auth code with error: ${error}`;
        }
    </script>
</body>
</html>
```

You could host these static files using your favorite HTTP server. For quick stuff, I use [`http-server`](https://www.npmjs.com/package/http-server) from npm:

```bash
npx http-server ./ -p 5500
```

Now let's see what happens in the iframe. In an InPrivate/Incognito window, I navigate to `http://localhost:5500/` and am met with an error in the iframe:

{% include image.html file="2021-03-12-aad-iframe/aad-iframe-error.png" alt="An iframe with an error image and text saying login.microsoftonline.com refused to connect." %}

And opening the browser's dev tools, I find this error in the console:

```
Refused to display 'https://login.microsoftonline.com/' in a frame because it set 'X-Frame-Options' to 'deny'.
```

In other words, AAD has headers set to prevent their login page from being displayed in a frame. But those security risks really only apply if the user is being presented with a login prompt where they have to click a button or type their password. What if we could explicitly tell AAD *not* to show any prompts?

# Setting the `prompt` parameter

Turns out we can! There's an optional parameter called `prompt`. From the [AAD docs on the auth code flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow#request-an-authorization-code), the `prompt` parameter:

> Indicates the type of user interaction that is required.

And one of the possible values is `none`:

> ...it will ensure that the user isn't presented with any interactive prompt whatsoever. If the request can't be completed silently via single-sign on, the Microsoft identity platform will return an `interaction_required` error.

Let's try setting `prompt=none` in the login URL. In my `iframe` above, I add `&prompt=none` to the end of the `src` URL, so the full URL is:

```
https://login.microsoftonline.com/common/oauth2/v2.0/authorize
?client_id=ec33faa3-71b7-4675-a053-0ddfa673e40d
&response_type=code
&redirect_uri=http://localhost:5500/auth-end.html
&response_mode=query
&scope=openid%20offline_access%20https%3A%2F%2Fgraph.microsoft.com%2Fuser.read
&prompt=none
```

Refreshing my test page, I now see this error in the iframe:

```
Failed to get auth code with error: interaction_required
```

This might seem no better than before, but there's a big difference. Instead of getting a framing error, I'm now seeing an error from my `auth-end` page, meaning AAD successfully redirected to my page. I can confirm this by checking the network requests in dev tools. After the request to AAD's `/authorize` endpoint, there was a request to my `/auth-end` page:

```
http://localhost:5500/auth-end.html?error=interaction_required&error_description=...&state=12345
```

As the docs suggested, I'm getting an `interaction_required` error because the login request couldn't be completed silently. So the question is, when *can* the request be completed silently?

# Establishing an AAD session

Since I'm testing this in a private browser window, I don't have a signed-in session with AAD. Let's change that. One easy way is by going to `https://portal.office.com` (in the same browser window) and signing in with a work account.

Now if I refresh my test page, I see:

```
Successfully got auth code: 0.ASkAXKRA...
```

Success! Within the iframe, AAD was able (and willing) to silently use my existing session and redirect to my URL with an auth code. To finish the flow, I would have to exchange the auth code for an access token on the server. Or, if I was using a flow like auth code with PKCE or implicit, then I could get a token on the client.

# Gotchas

There are a couples cases I've found where you'll still get an `interaction_required` error, even with an existing AAD session:

- **Consent.** When I went through the auth flow once in the setup step, my user provided consent to the app. If this had been a brand new app, or if I had signed in with a different user, AAD would require the user to provide consent, which requires user interaction. So for a user's first sign-in, you would have to fall back to traditional methods like popups. Or, if it makes sense for your app, you could use [admin consent](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-admin-consent) to prevent users from ever having to provide consent.
- **Multiple accounts.** If you use multiple accounts, AAD will often show an account selection screen so that you can choose which account to sign in with. When signing in with `prompt=none`, AAD can't show that screen, and it won't try to guess which account you want. However, you can give AAD a hint via the `login_hint` or `domain_hint` parameters (see the [AAD docs on the auth code flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow#request-an-authorization-code)). If the hint is enough for AAD to disambiguate the desired account, then the flow can succeed without user interaction.

Admittedly, this isn't useful in a lot of cases since several conditions have to line up: the user needs an existing session with AAD, the app needs to get consent ahead of time, and your app needs a domain hint for the user. But it at least gives us insight into how libraries like MSAL.js are able to acquire/renew tokens in "hidden iframes" and dispels the notion that the login page absolutely can't exist in an iframe.