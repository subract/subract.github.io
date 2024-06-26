---
title: "False security: Dashy's client-side authentication"
date: 2024-03-27T12:16:00-06:00
---

**Update 3/28**: The devs have announced that the auth system is to be deprecated. See details [below]({{< ref "#a-positive-change" >}} "update").

About a month ago, I went looking for a dashboard for my [homelab](https://github.com/subract/ansible)—something to help visualize the services I run. I found [Dashy](https://dashy.to/), a popular (14.6k GitHub stars) dashboard designed for self-hosters. I deployed it and started configuring it, but noticed that something about its authentication felt *off*. I started digging and quickly found its security to be borderline useless, permitting unauthenticated reads *and writes* of its configuration. I'm sharing my findings here with the goals of encouraging self-hosters to think critically about their apps and to encourage developers to document their app's security with care.



## Dashy who?

![Dashy demo screenshot](dashy-demo.png#center)

Above: screenshot of Dashy’s [official demo instance](https://demo.dashy.to/)

A quick primer: Dashy's an "open source, highly customizable, easy to use, privacy-respecting dashboard app." Dashy provides an extensive set of features for its purpose, including status monitoring, widgets, customizable layouts, and—most importantly for our discussion today—**authentication**.

On Dashy's [main README](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/README.md#authentication-), it claims the following:

> Dashy has full support for secure single-sign-on using [Keycloak](https://www.keycloak.org/) for secure, easy authentication, see [setup docs](https://github.com/Lissy93/dashy/blob/master/docs/authentication.md#keycloak) for a full usage guide. There is also a basic auth feature, which doesn't require additional setup.

Dashy supports multiple users per instance and provides [granular access control](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#granular-access) to dashboard pages and items. Dashy's docs encourage users to store a variety of information in their dashboards, ranging from semi-private to highly sensitive:

- Photos in the [Images widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#image)
- "Private" links to hosted files, notes, pastes, videos, etc.
- App credentials and API keys:
    - The [Nextcloud widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#nextcloud-user) requires storing a username and app password, which provides access to all user data
    - The [Drone CI widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#drone-ci-builds) requires storing a Drone CI API key, which provides access to all Drone functionality for the user
    - The [AnonAddy widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#anonaddy) requires storing a AnonAddy (now addy.io) API key, which provides access to create and forward email addresses for the user


## Bypassing Dashy's access control

{{< box info >}}
**Get the flag**

**Update 4/19**: I've **decommisioned** my demo instances, since they've pretty much run their course. You can always download Dashy yourself to test out the exploits!

Before we get into why Dashy's authentication doesn't protect your dashboard in the way you might expect, I offer a challenge for the curious:

I've stood up two demo instances, both running the latest version as of the time of writing (v2.1.2, Mar 27, 2024). Dashy's [built-in authentication](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#built-in-auth) protects the first, while [its Keycloak integration](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#keycloak) protects the second.

Each has a flag hidden on the dashboard. Feel free to poke around and try to recover the flags before going further. The rest of this article has *spoilers*. 

I ask that you not attempt to exploit my demo instances further than reaching the flags. Feel free to spin up your own server for that.

- [~~Demo 1: Built-in auth~~](https://dashy-demo.subract.dev)
- [~~Demo 2: Keycloak~~](https://dashy-demo-keycloak.subract.dev)
{{< /box >}}

In brief, Dashy's access control works like this:

- Load the app
- Request the app configuration from the server
- If authentication is configured, present an authentication page to the user
    - With built-in authentication, present the user a simple login form
    - With Keycloak authentication, redirect the user to the Keycloak sign-in page
- If the user authenticates successfully, present the user with their personal dashboard

The limitation in how Dashy's access control works is the fact that it's **entirely client-side**, running in the user's browser. That means it's vulnerable to tampering using the  Javascript console. The server doesn't participate in the process. Whether Dashy uses Keycloak or not is entirely irrelevant to the security, as the examples below show. 

The total lack of server involvement renders the authentication and access control **fundamentally broken**. I'd identify it as instances of [CWE-922: Insecure Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/922.html) and [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html), weaknesses that fall under categories [one](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) and [seven](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/) of the 2021 OWASP Top 10. In short, the authentication page is security theater that can be trivially bypassed.

### The hard way: Javascript tomfoolery

 Myriad ways to bypass Dashy's authentication exist. If you poked at my demo instance, perhaps you've found your own already. I cobbled together crude examples for each of my demo instances:

- For [Demo 1: Built-in auth](https://dashy-demo.subract.dev):
    - Open a browser console
    - Set a breakpoint on `Auth.js:28`, reload the page and run till it hits the breakpoint
    - Run `auth.users = []` in the browser console
    - Continue execution to reach the dashboard
- For [Demo 2: Keycloak](https://dashy-demo-keycloak.subract.dev)
    - Open the browser console
    - Set a breakpoint on `KeycloakAuth.js:9`, reload the page and run till it hits the breakpoint
        - Due to the automatic redirect to Keycloak, this may be tricky
    - Run `config.appConfig.auth = {}` in the browser console
    - Continue execution to reach the dashboard

The steps here are incidental—the point is to show that any form of client-side authentication like this **provides no meaningful security**. Disabling Webpack source maps wouldn't provide more security, just more obscurity. Worse, this sort of browser meddling isn't needed. There's a much easier way.

### The easy way: view the config directly

Recall the second step in Dashy's authentication process from before:

> Request the app configuration from the server

Here's the thing: an attacker doesn't *need* to view your dashboard to get your secrets, because the *full config file is available with no authentication*. If you watch your browser's network requests, you can see it pull `/conf.yml` from the server. You can request the file directly: here's links to the config files for [demo 1](https://dashy-demo.subract.dev/conf.yml) and [demo 2](https://dashy-demo-keycloak.subract.dev/conf.yml). See the flags? They could just as easily be powerful API keys.

### Bonus points: change the config

In a similar fashion, an attacker can change Dashy's configuration without authentication. While Dashy provides [a configuration option](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/configuring.md#appconfig-optional)  (`preventWriteToDisk`) to prevent users from changing the configuration of a Dashy instance, this setting is—you guessed it—*only checked on the client*. There's nothing stopping an attacker from CURLing the `/config-manager/save` endpoint with arbitrary configuration for the app.

The ability for an unauthenticated user to change the config of the dashboard opens up entirely new security implications. An attacker can delete the configuration for a denial-of-service attack (though a built-in backup mechanic makes this somewhat less effective). Far worse, an attacker can *redirect dashboard links* to credential-stealing phishing sites. A user is unlikely to notice their personal dashboard sending them to a malicious URL.

An admin could apply various server-side mitigations to prevent any config changes (making the config file immutable is a good start), but doing so neuters a large fraction of the app's functionality, as editing the config from the GUI becomes impossible. On my demo instances, I made the file read-only to prevent modification.

## Mixed messages

I believe Dashy has both a *security problem* and a *communication problem*. Not every app needs to provide strong authentication on its own. Other ways to secure a web app exist, like reverse proxies tied in with auth providers that can provide strong authentication before allowing user to access the app in the first place.

To its credit, the Dashy docs [recommend several good options](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#alternative-authentication-methods) for securing a Dashy instance, but they're buried under "Alternative Authentication Methods" on the Authentication documentation page.

Indeed, digging further into Dashy's documentation reveals a wealth of knowledge on securing a Dashy deployment. The app management page has [an extensive section](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/management.md#container-security) on securing a containerized Dashy deployment. The privacy and security page has a [section](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/privacy.md#security-features) on the app's security features and links to the primary author's excellent [Personal Security Checklist](https://github.com/Lissy93/personal-security-checklist). The care given to security in many parts of the app makes the paper-thin authentication an anachronism.

A user glancing over the README could be misled into thinking that Dashy's authentication is enough to protect their secrets. They might further believe that using Keycloak provides more security, as is [directly stated](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#keycloak) in the docs:

> Dashy also supports using a [Keycloak](https://www.keycloak.org/) authentication server. The setup for this is a bit more involved, but it gives you greater security overall, useful for if your instance is exposed to the internet.

A further quote from the [privacy and security page](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/privacy.md#security-features) reads:

> If your dashboard is exposed to the internet and/ or contains any sensitive info it is strongly recommended to configure access control with Keycloak or another server-side method.

This suggestion is downright irresponsible. The fact that Keycloak does authentication on the server is moot when it's called solely from a client-side library.

### A positive change

Before publishing this post, I wrote an email to [the repo's security contact email](https://github.com/Lissy93/dashy/security) outlining my concerns and offering to put together a pull request with some proposed changes.

Four hours later, [a new commit](https://github.com/Lissy93/dashy/commit/0894f71056d50cd787a532312691b8e488c05faa) appeared adding a prominent disclaimer to the authentication documentation page noting that the built-in auth isn't intended to protect instances on the Internet and directing users to use a reverse proxy or VPN.

I applaud this change, but I don't think it goes nearly far enough. The main README is still the same, as are the other docs cited above. I didn't receive a reply to my email or to a follow-up sent 11 days later.

{{< box info >}}
**Update 3/27**

I've just now found [an existing issue](https://github.com/Lissy93/dashy/issues/668) on the repo that raises these issues, posted in 2022. The primary dev asserts they're aware of the issue and hoped to address it in v2.1.2, which has already been released. I'll update this post further if/when those changes are completed. Until then, the issues raised and my recommendations stand.

**Update 3/28**

The key developer has announced the **deprecation of built-in auth entirely**, which really resolves all my converns once implemented. This announcement was made on **Feb 22**, six days after my initial email was sent, but since it's rather buried in the GitHub discussions (and my email received no reply), I had no idea until [a maintainer finally dropped me a lnk](https://github.com/Lissy93/dashy/issues/668#issuecomment-2024390223). 

This is exactly the change I hoped to see. Again, I'll update this article if/when the changes are implemented.
{{< /box >}}

## Recommendations

### …to Dashy users

I'd recommend using another dashboard. [Homepage](https://gethomepage.dev) is an excellent alternative. It supports many of Dashy's flashy features like widgets and status monitoring, but—unlike Dashy—it holds the API keys for these services on the server, never providing them to the client. Homepage further offers no built-in authentication, leaving the task to purpose-built apps.

If you must use Dashy, don't rely on Dashy's authentication to protect any non-public information. Don't expose Dashy to the internet if it contains *any* non-public information—and then only if you can secure the configuration against modification. Use extra caution when storing API keys in widget configuration. Use alternative authentication mechanisms like reverse proxies.

### …to Dashy developers

At a minimum, the project documentation should receive a thorough update noting the limitations of its authentication scheme. Promote alternatives like reverse proxies to primary solutions.

For a better solution, I'd advocate for removing the authentication system entirely and replacing it with a list of profiles. Users select their desired profile from a list, Netflix-style. This offers the per-user customization benefits without the security theater of client-side authentication.

### …to app developers

Be cautious about the way you present your app's security properties to users. Be transparent about what your app can and can't do on it's own. Offer suggestions about how to protect your app. Don't put your users in a situation where they might believe their data is protected when it isn't.

### …to self-hosters

Choose your apps with care. Use purpose-built apps like reverse proxies and authentication servers to protect your apps. Practice defense-in-depth and think about how your security would look if any given layer failed. Exercise extreme caution when exposing apps to the Internet. 

Above all, don't trust that an app is secure just because it has a login page.

## Timeline

**February 16, 2024**: discovered configuration exposure vulnerability

**February 16, 2024**: initial notification sent to security@mail.alicia.omg.lol

**February 27, 2024**: discovered configuration write vulnerability

**February 27, 2024**: follow-up email sent

**March 27, 2024**: no replies to any emails received. Blog post published.