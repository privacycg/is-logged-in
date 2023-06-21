
# Explainer: Login Status API

A [Work Item](https://privacycg.github.io/charter.html#work-items)
of the [Privacy Community Group](https://privacycg.github.io/).

## Editors:

- (suggestion) [John Wilander](https://github.com/johnwilander), Apple Inc.
- (suggestion) [Ben Vandersloot](https://github.com/bvandersloot), Mozilla
- (suggestion) [Sam Goto](https://github.com/samuelgoto), Google Inc.

## Participate
- https://github.com/privacycg/login-status-api/issues

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## Introduction

This explainer proposes a Web Platform API called the **Login Status API** which websites can use to inform the browser of their users's login status, so that other Web APIs can operate with this additional signal.

On one hand, browsers don't have any built-in notion of whether the user is logged in or not.  Neither the existence  of cookies nor frequent/recent user interaction can serve that purpose since  most users have cookies for and interact with plenty of websites they are not logged-in to. High level Web Platform APIs, such as form submission with username/password fields,  WebAuthn, WebOTP and FedCM, can (implicitly) record login, but don't record logout - so the browser doesn't know if the user is logged in or not.

On the other hand, there is an increasing number of Web Platform APIs (e.g. [FedCM](https://github.com/privacycg/is-logged-in/issues/53), [Storage Access API](https://github.com/privacycg/storage-access/issues/8)) and Browser features (e.g. visual cues in the url bar, freeing up long term disk and backup space) that could perform better under the assumption of whether the user is logged in or not.

This proposal aims at creating a set of **extensible** APIs (e.g. HTTP headers, JS APIs, and Cookie annotations) to manage a deliberate, explicit, **opted-in** and **self-declared** signal that websites can use to inform the browser.

Because of the self-declared property of the signal, Web Platform APIs and Browser features **must** design their use with **abuse** in mind.

## Proposal

Below we present a proposal for how a web API for logged in status could look and work. This is a starting point for a conversation, not a fully baked proposal.

### Representation and Storage

The website's **self-declared** login status is represented as a single bit **per origin** with the following possible values:

- `unknown`: the browser has never observed a login nor a logout
- `logged-in`: a user has logged-in to **an** account on the website
- `logged-out`: the user has logged-out of **all** accounts on the website

 By **default**, every origin has their login status bit set to `unknown`.

The login status bit represents the **client-side** state of the origin in the browser and can be out-of-date with the **server-side** state (e.g. a user can delete their account in another browser instance). Even then, due to cookie expiration, it is an imperfect representation of the client-side state, in that it may be outdated.

### Setting the Login Status

There are several mechanisms that a website can use to set the login status:

> TODO: figure out if we can put these behind user activation.

#### JS API

Here’s how the API for recording a login / logout could look:

```javascript
partial interface Navigator {
  Promise<void> setLoggedIn(optional LoginStatusOptions options);
  Promise<void> setLoggedOut(optional LogoutStatusOptions options);
};

interface LoginStatusOptions {
  // To be extended by specific web platform features
}

interface LogoutStatusOptions {
  // To be extended by specific web platform features
}
```

#### HTTP API

Here is how a HTTP server could operate on the signals:

```javascript
Sign-in-Status: type=idp, action=signin
Sign-in-Status: type=idp, action=signout-all
```

#### Cookie API

> It is possible that we may want to annotate cookies (or [HTTP State
Tokens](https://mikewest.github.io/http-state-tokens/draft-west-http-state-tokens.html)) so that they can convey the signal. We don't know what that looks like or whether it is needed at all, but it seemed important to acknowledge that Cookies may be an important part of API design.

### Using the Login Status

The login status is a bit that is available to Web Platform APIs and Browser features outside of this specification. Specifications can extend the API to gather more specific signals needed.

#### Examples

##### FedCM

[FedCM](https://github.com/privacycg/is-logged-in/issues/53#issue-1664953653) needs a mechanism that allows Identity Providers to signal to the browser when the user is logged in. It extends the Login Status API to give them that API surface.

```javascript
// Records that the user is logging-in to a FedCM-compliant Identity Provider.
navigator.setLoggedIn({
  idp: true
});
```

##### Storage Access API

The [Storage Access API](https://github.com/privacycg/storage-access/issues/8#issue-560633211) needs a mechanism that allows websites to signal to the browser when the user is logged in. It may use the raw signal or extend it in case it needs more information:

```javascript
// Records that the user is logging-in, which allows the Storage Access API to conditionally dismiss its
// UX.
navigator.setLoggedIn();
```

##### Browser Status UI

Much like favicons, websites may be benefited from having a login status indicator in browser UI (e.g. the URL bar). The indicator could extend the login status API to gather the explicit signals that it needs from the website to display the indicator (e.g. a name and an avatar).

```javascript
// Records that the user is logging-in to a FedCM-compliant Identity Provider.
navigator.setLoggedIn({
  profile: {
    name: "John Doe",
    picture: "https://website.com/john-doe/profile.png",
  }
});
```

#### The Assumption of Abuse

Every user of the login-status bit **MUST** assume that the login-status bit is:

1. Self-declared: any website can and will lie to gain any advantage
2. Client-side: the state represent the website's client-side knowledge of the user's login status, which is just an approximation of the server-side's state (which is the ultimate source of truth) 

One potential for abuse is if websites don’t call the logout API when they should. This could allow them to maintain the privileges tied to login status even after the user logged out.

Features using the Login Status bit need to assume that (1) an (2) are the case and design their security and privacy models under these conditions.

## Challenges and Open Questions

## Considered alternatives

### An API-specific Signal

One obvious alternative that occurred to us was to build a signal that is specific to each API that is being designed. Specifically, [should FedCM use its own signal or reuse the Login Status API](https://github.com/privacycg/is-logged-in/issues/53)?

While not something that is entirely ruled out, it seemed to most of us that it would be worth trying to build a reusable signal across Web Platform features.

### Implicit Signals

Another trivial alternative is for Web Platform APIs to implicitly assume the user's login status based on other Web Platform APIs, namely username/password form submissions, WebAuthn, WebOTP and FedCM.

While that's an interesting and attractive venue of exploration, it seemed like it lacked a few key properties:

- first, logout isn't recorded by those APIs, so not very reliable
- second, the login isn't explicitly done and the consequences to other Web Platform APIs isn't opted-into. 
- third, username/passwords form submissions aren't a very reliable signal, because there are many ways in which a browser may be confused by the form submission (e.g. the password field isn't marked explicitly so but rather implemented in userland)

So, while this is also not an option that is entirely ruled out, a more explicit signal from developers seemed more appropriate.

### User Signal

Another alternative that we considered is an explicit user signal, likely in the form of a permission prompt. While that would address most of the abuse vectors, we believed that it would be too cumbersome and hard to explain to users (specially because the benefits are in the future).

## Stakeholder Feedback / Opposition

There is an overall directional intuition that something like this is a useful/reasonable addition to the Web Platform by the original proposers of this API and the APIs interested in consuming this signal (most immediately [FedCM and SAA](https://github.com/privacycg/is-logged-in/issues/53)).

This is currently in an extremely early draft that is intended to gather convergence within browser vendors.

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge
them in an ongoing way! It helps build community and, as we only get by
through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in
alphabetical order.]

Former editor: [Melanie Richards](https://github.com/melanierichards), Microsoft
