---
permalink: index.html
layout: post
title: WebID
---

**TL;DR**; This is an **early exploration** of ways that Browsers can help users make [safer](#the-rp-tracking-problem) decisions about what data from identity providers they share with websites. 

This explainer is broken down into:

- [The Problem](#the-problem)
- [Prior Art](#prior-art)
- [An early exploration](#early-exploration)
- [Related Problems](#related-problems)

# The Problem

Over the last decade, identity federation has unquestionably played a central role in raising the bar for authentication on the web, in terms of ease-of-use (e.g. passwordless single sign-on), security (e.g. improved resistance to phishing and credential stuffing attacks) and trustworthiness compared to its preceding pattern: per-site usernames and passwords.

The standards that define how identity federation works today were built independently of the web platform, and their designers had to work **around** its limitations rather than extending them. Because of that, existing user authentication flows rely on general web capabilities such as top-level navigation, link decoration, window popups and cookies.

Because these general purpose primitives can be used for an open ended number of use cases (some of them increasingly allowing users to be tracked), browsers have to apply policies that capture the **lowest common denominator**, at best applying cumbersome permissions (e.g. popup blockers) but at worst entirely [blocking](https://www.chromium.org/Home/chromium-privacy/privacy-sandbox) them.

If the low level primitives are being abused for tracking purposes and browsers are applying stricter policies around them, how do we keep identity federation around?

## The classification problem

Some of the problem starts with what we have been calling the classification problem.

When federation was first designed, it was rightfully designed **around** the existing capabilities of the web, rather than **changing** them. Specifically, federation worked with callbacks on top of **redirects** or **popup windows**, which didn't require any redesign, redeployment or negotiation with browser vendors.

These **general purpose** primitives enabled a variety of use cases, which include, among other things, federation. However, they also enable cross-site communication, which isn't always consented or acknowledged by users, namely via [decorating links](https://www.chromium.org/Home/chromium-privacy/privacy-sandbox).

Because the cross-site communication takes place in a general purpose medium, it is hard for browsers to tell the difference between cross-site communication that is used for exchanging identity data or other cases where intervention is needed.

The classification problem is the problem of taking the existing federation mechanisms built on top of general purpose primitives and classify them such that they can be told apart.

![](static/mock9.svg)

For example, IDPs use full page redirects which take the user out of context of the site they were trying to use. IDPs also try to use the general purpose pop-up window, but often because browsers are unaware of the use federation makes of popups, it has to apply a general rule across all usages, often blocking an IDP popup that would be otherwise helpful.

![](static/mock11.svg)

To solve the classification problem, the following needs to hold:

- The user agent should be able to distinguish between authentication and tracking
- The affordance should be applicable/meaningful if and only if it is used for authentication (i.e. cannot be abused outside of its scope)

If the classification problem was solved alone it would delineate the boundaries and applications of the affordances, but it wouldn't necessarily move the needle in making sure that the users are making safer choices.

In order to prevent tracking in federation, one has to solve what we've been calling the [RP tracking problem](#the-rp-tracking-problem) in federation and the [IDP tracking problem](#the-idp-tracking-problem).

## The RP Tracking problem

Relying party tracking is enabled through federation when services that the user signs in to **collude** with each other and other entities to deterministically or probabilistically **link** their user's accounts to build and get access to a richer user profile (e.g. one site selling data on browsing history for ads targeting to another service). While this could be enabled without federation per se (user could manually provide a joinable email address or phone number), federated identity providers have an opportunity to address this problem by providing the user with service-specific data and directed identifiers. 

![](static/mock3.svg)

## The IDP Tracking problem

Even if identity providers were to provide site-specific data and directed identifiers, IDPs and RPs can exchange data without the user explicitly being aware of what information is flowing between the parties, and that the IDP may have insight into the user’s activity across sites. Federation is implemented via parameters on redirects / top level navigation, which allow for arbitrary data exchange, without insight or controls by the user’s browser.

![](static/mock10.svg)

# Prior Art

By far, the closest analogy to this work is the great work of [BrowserID](https://github.com/mozilla/id-specs/blob/prod/browserid/index.md#web-site-signin-flow) during the [Mozilla Personas](https://developer.mozilla.org/en-US/docs/Archive/Mozilla/Persona/The_navigator.id_API) effort (postmortem [here](https://wiki.mozilla.org/Identity/Persona_AAR)). In many ways, the goals that BrowserID was trying to achieve as well as the mechanisms that were created are a lot alike what’s being proposed here. There are significant differences in strategy and design, but let’s start with the similarities first because there are many.

First, BrowserID was clearly trying to solve similar problems, namely IDP Tracking and friction. The mechanisms that were created clearly had [strong builtin privacy components](https://www.youtube.com/watch?v=qHpFwdQw2wQ) to prevent IDPs from knowing about RPs and vice versa, via the chain of signatures gathered from IDPs (mainly email providers) and certificates generated by the browser.

Second, from a mechanism perspective, there was clearly a separation between an RP to Browser API as well as a Browser to IDP API. 

```javascript
navigator.id.get((assertion) => {}, {
  backgroundColor: "#606B72",
  siteName: "My Example Site"
});
```

The [RP calls a browser native API](https://github.com/mozilla/id-specs/blob/prod/browserid/index.md#web-site-signin-flow) to fetch an assertion which gets mediated by the [browser in coordination with the IDP](https://github.com/mozilla/id-specs/blob/prod/browserid/index.md#identity-provisioning-flow).

```javascript
// set up UI
navigator.id.beginAuthentication(function(email) {
  // update UI to display the email address
});
```

The postmortem analysis [here](https://wiki.mozilla.org/Identity/Persona_AAR) is very insightful in understanding what were the challenges faced and gives this proposal a solid place to work from. In many ways, we think some of these insights are rooted in the observation we made earlier about backwards compatibility with RPs and user’s current behavior, which we are deliberately trying to avoid. 

# Early Exploration

There is a wide set of privacy and usability goals for identity sharing on the web, but early on we ran into better understanding the structural deployment of federation on the web, specifically the properties that make different strategies more or less plausible.

For example, it is clear that there are relatively few public [IDPs](#idp) in use (say, tens), particularly in comparison to the number of [RPs](#rp) (say, millions) and their users (say, billions). A structural change that only requires adoption by IDPs and no changes or engagement on the part of RPs and users is significantly easier compared to redeploying millions of RPs or retraining billions of users.

Fortunately, in more cases than not, RPs implement federated identity importing a script provided by - and under the control of - IDPs, giving us a major deployment vehicle: IDP SDKs loaded into RPs. 

![](static/mock7.svg)

Nonetheless, while much of the client side code is under the (few) IDPs to control (e.g. we can replace redirects by other means), all of the server side code is under the (many) RPs to control, meaning that that is significantly harder to change (say, years). The cases where RPs implement federated identity without a dynamically loaded SDK will have a longer deployment window and will be discussed separately. 

Likewise, changing user behavior and norms is hard because of the number of people involved (say, billions). Unfamiliar login flows could result in users declining to use federated options, and instead opting for username/password credentials during RP account creation. To address that, this proposal aims to provide an experience that minimizes the divergence from existing federated identity user experience as much it possibly can (e.g. introducing new user decisions to be made).

So, with this deployment constraint in mind, let's look at what could be done.

## Browser API

Currently, sign-in flows on websites begin with a login screen that provides the user options to use federated identity, as illustrated in the mock below. Today, clicking the button for an IDP relies on general purpose primitives (typically [redirects or popups](#low-level)) to an IDP sign-in flow. 

![](static/mock1.svg)

In this formulation, we provide a [high-level](#high-level), identity-specific API that allows browsers to [classify](#the-classification-problem) the otherwise **opaque** transactions that are enabled by [low-level](#low-level) APIs.

By classifying as an identity data exchange, browsers are now in a position to provide domain specific guidance to users regarding the consequences of the specific identity transaction.

There is a spectrum of **positions** to be taken here, but as a starting point, take as an example the most strict of the spectrum where the browser fully mediates the sign-in flow and takes full responsibility over the data exchange:

![](static/mock14.svg)
We can break these down in four stages:

- The [triggering](#the-triggering-stage) stage: the user makes a user gesture and the RP calls a newly introduced API
- The [provisioning](#the-provisioning-stage) stage: the browser makes an assessment over the risks involved in exchanging the data (e.g. does it contain a global identifier?),
- The [consent](#the-consent-stage) stage: makes sure the user consents and understands the risks involved (e.g. scarier prompts when a global identifier is found) and
- The [creation](#the-creationg-stage) stage: how much the identity provider knows about the relying party.

Lets go over each of these stages in more detail.

#### The Triggering Stage

In the triggering stage, the [low-level](#low-level) redirect/popup flow gets replaced by the invocation of a new **high-level** identity-specific API that enables RPs to request IdTokens, for example:

```javascript
// This is just a possible starting point, largely TBD.
let {idToken} = await navigator.credentials.get({
  provider: "https://accounts.example.com",
  // other OpenId connect parameters
});
```

It is too early to discuss the specifics of the API, here is a declarative formulation that could potentially work too:

```html
<input type=”idtoken” provider=”https://accounts.example.com”>
```

At this stage, the browser makes an assessment of the user's intention, for example making sure that the API was used as a result of a user gesture.

#### The Provisioning Stage

The browser then proceeds to talk to the IDP (e.g. via a **.well-known** convention) and gather the user's identity token and what properties it holds (e.g. is this a directed identity?). The IDP also makes claims about its policies in protecting user's data:

The browser intermediates the data exchange according to its assessment of the privacy properties involved: the more it believes that the exchange is respecting the user's privacy the less it has to raise the user's awareness of the perils involved (e.g. scary permission prompts). 
 
We believe a combination of strategies are going to be involved, but it seems hard to escape some form of agreement on policy, specifically because of server-side / out-of-band collusion where browsers aren't involved. So, as a starting point, this strawman proposal starts with a mechanism and convention that allows IDPs to explicitly acknowledge certain service agreements.

```js
// Available on a .well-known/webid file:
{
  "@context": "https://www.w3.org/ns/webid",
  "@type": "IdentityProvider",
  "policies": [
    "https://tbd.org/policies/privacy/1.0"
  ]
  ... TBD ...
  // possibly signed by a neutral authority that verifies the claims?
}
```

The IDPs host the `.well-known/webid` file to acknowledge and express agreement on privacy policies. Browsers load the file and inform the user accordingly.

Importantly, the browser doesn't unveil who the RP is quite yet, without the user's permission. So, a token is created but not quite yet signed by the IDP:

```json
{
 "iss": "https://accounts.google.com",
 "sub": "110169484474386276334",

 "directed": "true",

 "name": "Sam G",
 "given_name": "Sam",
 "family_name": "G",
 "email": "sjkld2093@gmail.com",
 "email_verified": "true",
}
```

### The Consent Stage

With the user's identity information at hand, the browser then proceeds to gathering consent from the user and raising awareness of any peril that may be involved according to the assessment it made in the last stage.

![](static/mock13.svg)

### The Creation Stage

After the user consents, the browser can now be confident about the user's intention and finally unveils to the IDP the RP, which the IDP can then use to mint a new token addressed/directed to the specific RP.

```json
{
 "iss": "https://accounts.google.com",
 "sub": "110169484474386276334",
 "aud": "https://example.com",

 "name": "Sam G",
 "given_name": "Sam",
 "family_name": "G",
 "email": "sjkld2093@gmail.com",
 "email_verified": "true",
}
```

The IdToken is then returned back to the RP which can effectively get the user logged in.

# Related Problems

## The [NASCAR flag](https://developers.google.com/identity/toolkit/web/federated-login#the_nascar_page) problem

Every website has a different sign-in process and has to show a list of supported identity providers for the users to choose. The user is left to determine which identity provider to use, which one they may have used last time, what might happen if they pick a different IDP this time, and what what data might get shared, typically without any support from the browser in remembering the user’s past choice or highlight relevant options. We believe that, by pulling some of the responsibility for the browser, we can offer a personalized IDP disambiguation UI which can lead to higher conversion rates, but yet maintain user privacy.

![](static/mock12.svg)

Although not directly related to federation per se, there exist a number of other authentication and identity related problems that are worth mentioning, which an be addressed by other efforts that may be related to, but pursued independently of efforts to improve federation.

## Identity Attribute Verification

Verifying phone numbers and emails is tedious: currently, verification is often done manually by users without assistance from the browser or IDP. For example, to verify email addresses a service typically sends an OTP (one-time code) to the user’s inbox to be copied/pasted. Similarly, for phone numbers, an SMS message is sent to the user’s phone to be copied/pasted too. There are clear ways here where the browser can step in to help (e.g. [WebOTP](https://github.com/WICG/WebOTP)), and it would generally preferable for authoritative identity providers to assert these attributes wherever possible.

## Cross device sign-in

Because cookies are not propagated across devices, a user has to sign back in (and remember account info, etc) on new devices. Often they end up having to go through a recovery flow, creating a duplicate account, or abandoning completely. Identity providers play an important role in facilitating cross-device sign-in, but we may be able to solve this more generally for user irrespective of their chosen authentication mechanism by expanding on web platform functionality such as the [Credential Management API](https://www.w3.org/TR/credential-management-1/).

## The Session State Opacity Problem

Because session state management is implemented via general purpose low level primitives (namely, cookies), when users intend to “log-out” there are no guarantees that anything necessarily happens (e.g. the origin can still know who you are, but it can pretend it doesn’t). Only clearing all cookies currently guarantees that an origin is not **adversarially tracking** you post log-out. There are proposals such as [IsLoggedIn](https://github.com/WebKit/explainers/tree/master/IsLoggedIn) to address this issue.

![](static/mock5.svg)

# Related Work

- [Building a More Private Web](https://blog.chromium.org/2020/01/building-more-private-web-path-towards.html)
- [Personas](https://wiki.mozilla.org/Identity/Persona_AAR) and [navigator.id](https://developer.mozilla.org/en-US/docs/Archive/Mozilla/Persona/The_navigator.id_API)
- [WebOTP](https://github.com/WICG/WebOTP)
- [Credential Manager](https://w3c.github.io/webappsec-credential-management/#federated)
- Add your work [here](https://github.com/samuelgoto/WebID/issues/new)

# Terminology

## RP

Relying Parties.

## IDP

Identity Providers.

## low level

General Purpose APIs, namely redicts and iframes.

## high level

Domains Specific APIs.
