---
title: "False security: Dashy's client-side authentication"
date: 2024-03-26T14:28:53-06:00
---

In this article, I'll present an analysis of [Dashy](https://dashy.to/), a relatively popular (14.6k GitHub stars) personal dashboard designed for self-hosters. We'll examine the (lack of) security with its authentication scheme that leads to unauthenticated users being able to read *and write* to its configuration, but just as importantly, we'll take a look at *how it communicates about its security*. My aim is to encourage users to to think critically about how the apps they use are protected, and to encourage developers to think carefully about how they communicate their application's security guarantees to users.

## Dashy who?

![Dashy demo screenshot](dashy-demo.png#center)

Screenshot of Dashy’s [official demo instance](https://demo.dashy.to/)

A quick primer on Dashy: It's an "open source, highly customizable, easy to use, privacy-respecting dashboard app." Dashy provides an extensive set of features for its purpose, including status monitoring, widgets, customizable layouts, most importantly for our discussion today, **authentication**.

Here's what Dashy says about its authentication features on the [main README](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/README.md#authentication-):

> Dashy has full support for secure single-sign-on using [Keycloak](https://www.keycloak.org/) for secure, easy authentication, see [setup docs](https://github.com/Lissy93/dashy/blob/master/docs/authentication.md#keycloak) for a full usage guide. There is also a basic auth feature, which doesn't require additional setup.

Dashy supports multiple users per instance, and provides [granular access control](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#granular-access) to dashboard pages, sections and items. Users are encouraged to store a variety of bits of information in their dashboards, ranging from highly sensitive to semi-private:

- Application credentials
    - The [Nextcloud widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#nextcloud-user) requires storing a username and app password, which provides access to all user data
    - The [Drone CI widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#drone-ci-builds) requires storing a Drone CI API key, which provides access to all Drone functionality for the user
    - The [AnonAddy widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#anonaddy) requires storing a AnonAddy (now addy.io) API key, which provides access to create and forward email addresses for the user
- Photos, in the [Images widget](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/widgets.md#image)
- "Private" links to hosted files, notes, pastes, videos, etc.

## Bypassing Dashy's access control

{{< box info >}}
**Get the flag**

Before we get into why Dashy's authentication doesn't protect your dashboard in the way you might expect, I offer a challenge for the curious.

I've stood up two demo instances, both running the latest version as of the time of writing (Feb 27, 2024). One is protected by Dashy's [built-in authentication](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#built-in-auth), while the other is protected by [its Keycloak integration](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#keycloak).

Each has a flag hidden on the dashboard. Feel free to poke around and try to recover the flags before going further. The rest of this article has *spoilers*. 

I ask that you not attempt to exploit further than reaching the flags on my demo instances. Feel free to spin up your own server for that.

- [Demo 1: Built-in auth](https://dashy-demo.subract.dev)
- [Demo 2: Keycloak](https://dashy-demo-keycloak.subract.dev)
{{< /box >}}

In a nutshell, Dashy's access control works in the following fashion:

- Load the application
- Request the application configuration from the server
- If authentication is configured, present an authentication page to the user
    - With built-in authentication, the user is presented a simple login form
    - With Keycloak authentication, the user is redirected to the Keycloak sign-in page
- If the user successfully authenticates, present the user with their personal dashboard

The limitation in how Dashy's access control works is the fact that it is **entirely client-side**, running in the user's browser. That means it's vulnerable to tampering using the e.g. Javascript console. The server is uninvolved in the process. Whether or not the user chooses to use Keycloak is entirely irrelevant to the security, as the examples below demonstrate. It's security theater.

The total lack of server involvement renders the authentication and access control **fundamentally broken**. I'd identify it as instances of [CWE-922: Insecure Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/922.html) and [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html), weaknesses that fall under categories [one](https://owasp.org/Top10/A01_2021-Broken_Access_Control/) and [seven](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/) of the 2021 OWASP Top 10.

### The hard way: Javascript tomfoolery

There are myriad ways to bypass Dashy's authentication - if you poked at my demo instance, perhaps you've found your own already. I cobbled together crude examples for each of my demo instances:

- For [Demo 1: Built-in auth](https://dashy-demo.subract.dev):
    - Open a browser console
    - Set a breakpoint on `Auth.js:28`, reload the page, and run till it hits the breakpoint
    - Run `auth.users = []` in the browser console
    - Continue execution to reach the dashboard
- For [Demo 2: Keycloak](https://dashy-demo-keycloak.subract.dev)
    - Open the browser console
    - Set a breakpoint on `KeycloakAuth.js:9`, reload the page, and run till it hits the breakpoint
        - Due to the automatic redirect to Keycloak, this may take a few attempts, but is very doable
    - Run `config.appConfig.auth = {}` in the browser console
    - Continue execution to reach the dashboard

The steps here are largely incidental - the point is to demonstrate that any form of client-side authentication like this **provides no meaningful security**. Disabling Webpack source maps wouldn't provide more security, just more obscurity. Worse, though, this sort of browser meddling isn't needed - there's a much easier way.

### The easy way: view the config directly

Recall the second step in Dashy's authentication process from before:

> Request the application configuration from the server

Here's the thing: an attacker doesn't *need* to view your dashboard to get your secrets, because the *full config file is available with no authentication*. If you watch your browser's network requests, you can see it pull `/conf.yml` from the server. You can request the file directly - here's links to the config files for [demo 1](https://dashy-demo.subract.dev/conf.yml) and [demo 2](https://dashy-demo-keycloak.subract.dev/conf.yml). See the flags? They could just as easily be powerful API keys.

### Bonus points: Modify the config

In a similar fashion, an attacker can modify Dashy's configuration without authentication. While Dashy provides [a configuration option](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/configuring.md#appconfig-optional)  (`preventWriteToDisk`) to prevent users from modifying the configuration of a Dashy instance, this setting is - you guessed it - *only checked on the client*. There's nothing stopping an attacker from simply CURLing the `/config-manager/save` endpoint with arbitrary configuration for the application.

The ability for an unauthenticated user to modify the config of the dashboard opens up entirely new security implications. An attacker could, of course, simply delete the configuration (though a built-in backup mechanic makes this somewhat less effective). Far worse, though, an attacker could easily *redirect dashboard links* to credential-stealing phishing sites. A user is unlikely to notice they're being sent to the wrong URL from their personal dashboard.

There are various server-side mitigations that could be applied to prevent any config changes from being made (making the config file immutable is a good start), but doing so neuters a large fraction of the app's functionality.

## Mixed messages

I believe Dashy has both a *security problem* and a *communication problem*. Not every application needs to provide strong authentication! There are a multitude of ways to secure an web app - not exposing it to the Internet is a good start, and reverse proxies tied in with auth providers can provide strong authentication before a user is allowed to access the application in the first place.

To its credit, the Dashy docs [recommend several good options](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#alternative-authentication-methods) for securing a Dashy instance. Unfortunately, their buried under "Alternative Authentication Methods" on the Authentication documentation page.

Indeed, digging further into Dashy's documentation reveals a wealth of knowledge on securing a Dashy deployment. The app management page has [an extensive section](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/management.md#container-security) on securing a containerized Dashy deployment. The privacy and security page has a [section](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/privacy.md#security-features) on the app's security features, and links to the primary author's excellent [Personal Security Checklist](https://github.com/Lissy93/personal-security-checklist). The care given to security in many parts of the app makes the paper-thin authentication an anachronism.

A user glancing over the main readme could easily be misled into thinking that Dashy's authentication is sufficient to protect their secrets. They might further believe that using Keycloak provides additional security, as is [directly stated](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/authentication.md#keycloak) in the docs:

> Dashy also supports using a [Keycloak](https://www.keycloak.org/) authentication server. The setup for this is a bit more involved, but it gives you greater security overall, useful for if your instance is exposed to the internet.

A further quote from the [privacy and security page](https://github.com/Lissy93/dashy/blob/165809000840835720e444523a9e1c2eb478f1f9/docs/privacy.md#security-features):

> If your dashboard is exposed to the internet and/ or contains any sensitive info it is strongly recommended to configure access control with Keycloak or another server-side method.

Suggesting that using Keycloak for authentication provides sufficient security for putting a Dashy instance on the internet is downright irresponsible. The fact that Keycloak does authentication on the server is made moot when it's being called solely from a client-side library.

### A positive change

Before publishing this post, I wrote an email to [the repo's security contact email](https://github.com/Lissy93/dashy/security) outlining my concerns and offering to put together a pull request with some proposed changes.

Four hours later, [a new commit](https://github.com/Lissy93/dashy/commit/0894f71056d50cd787a532312691b8e488c05faa) was published adding a prominent disclaimer to the authentication documentation page, noting that the built-in auth was not intended to protect instances on the Internet from being accessed, and directing users to use a reverse proxy or VPN.

I applaud this change, but I do not think it goes nearly far enough. The main README has not been fixed, nor have other docs cited above. I did not receive a reply to my email, or to a follow-up sent 11 days later.

## Recommendations

### ...to Dashy users

Honestly, I'd recommend using another dashboard. [Homepage](https://gethomepage.dev) is an excellent alternative - it supports many of Dashy's flashy features like widgets, but unlike Dashy, it holds the API keys for these services on the server, never providing them to the client. Homepage further offers no built-in authentication, leaving the task to purpose-built applications.

If you really must use Dashy, do not rely on Dashy's authentication to protect any non-public information. Do not expose Dashy to the internet if it does not contain *only* public information - and then only if you can secure the configuration against modification! Be particularly careful when storing API keys in widget configuration. Use alternative authentication mechanisms like reverse proxies.

### ...to Dashy developers

At a minimum, the project documentation should receive a thorough update noting the limitations of its authentication scheme in the various places where authentication is discussed. Alternatives like reverse proxies should be promoted to primary solutions.

For a more complete solution, I'd advocate for removing the authentication system entirely and replacing it with a list of profiles. Users simply select their desired profile from a list, Netflix-style. This brings many of the customization benefits without the security theater of client-side authentication.

### ...to application developers

Be very cautious about the way you present your application's security properties to users. Be transparent about your application's limitations, and suggest practical ways to secure it. Don't put your users in a situation where they might believe their application is protected when it isn't.

### ...to self-hosters

Evaluate your applications carefully. Use purpose-built applications like reverse proxies and authentication servers to protect your applications. Practice defense-in-depth and think about how your security would look if any given layer failed. Exercise extreme caution when exposing applications to the Internet. 

Above all, don't trust that an application is secure just because it has a login page.

## Timeline

**16 Feb 2024**: Discovered that configuration can be read

**16 Feb 2024**: Initial notification sent to security@mail.alicia.omg.lol

**27 Feb 2024**: Discovered that configuration can be written as well as read

**27 Feb 2024**: Follow-up email sent

**26 Mar 2024**: No replies to any emails received. Blog post published.