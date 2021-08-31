# Explainer: IsLoggedIn

A [Work Item](https://privacycg.github.io/charter.html#work-items)
of the [Privacy Community Group](https://privacycg.github.io/).

## Editors:

- [John Wilander](https://github.com/johnwilander), Apple Inc.

## Former Editors:
- [Melanie Richards](https://github.com/melanierichards), formerly Microsoft

## Participate
- https://github.com/privacycg/is-logged-in/issues

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## Introduction

This explainer proposes an API called IsLoggedIn with which websites can
inform the web browser of the user's login state.

Currently, web browsers have no way of knowing if the user is logged in
to a particular website. Neither the existence of cookies nor
frequent/recent user interaction can serve that purpose since most users
have cookies for and interact with plenty of websites they are not
logged in to.

### Why Do Browsers Need To Know?

The current behavior of the web is “logged in by default,” meaning as
soon as the browser loads a webpage, that page can store data such as
cookies virtually forever on the device. That is a serious privacy issue
and also bad for disk and backup space. Long term storage should instead
be tied to where the user is truly logged in.

There could be other powerful features and relaxations of restrictions
besides storage that the web browser only wants to offer to websites
where the user is logged in.

The ability to do these things requires knowledge of where the user is
logged in.

## Existing Functionality

In olden times, Basic/Digest Authentication offered a way for browsers
to know where the user was logged in and help them to stay logged in.
However, there was never a way to log out. Regardless, those
technologies are now obsolete for many reasons. Today,
[WebAuthn](https://w3c.github.io/webauthn/) and password managers
(including the use of [Credential
Management](https://w3c.github.io/webappsec-credential-management/))
offer a browser-managed way to log in but those features neither cover
the expiry of the logged in session nor the act of logging out.

Cookies and other kinds of storage may carry login state but there is no
way to tell general storage and authentication tokens apart. Persistent
cookies have an expiry function which could serve as an automatic
inactivity logout mechanism whereas web storage such as IndexedDB
doesn’t even have an expiry functionality.

## Straw Man Proposal

Below we present a straw man proposal for how a web API for logged in
status could look and work. This is a starting point for a conversation,
not a fully baked proposal.

### API

Here’s how the API for setting IsLoggedIn to true could look:

```
navigator.setLoggedIn(
    username: non-whitespace string of limited length,
    credentialTokenType: “httpStateToken” OR “legacyAuthCookie”,
    optionalParams { }
) –> Promise<void>
```

The returned promise would resolve if the status was set and reject if
not. The API could potentially take an expiry parameter but here we’re
assuming that a designated [HTTP State
Token](https://mikewest.github.io/http-state-tokens/draft-west-http-state-tokens.html)
or “legacy auth cookie” manages the expiry of the login through their
own mechanisms.

Here’s how the API for setting IsLoggedIn to false could look:

```
navigator.setLoggedOut(optionalUsername) –> Promise<void>
```

The optional username parameter highlights that we might want to support
concurrent logins on the same website which would require the site to
keep track of who to log out and credential tokens to be scoped to user
names.

Here’s how the API for checking the IsLoggedIn status could look:

```
navigator.isLoggedIn() –> Promise<bool>
```

This last API could potentially be allowed to be called by third-party
iframes that do not currently have access to their cookies and website
data. The iframes may want to render differently depending on whether
the user is one of their logged in customers or not.

### Defending Against Abuse

If websites were allowed to set the IsLoggedIn status whenever they
want, it would not constitute a trustworthy signal and would most likely
be abused for user tracking. We must therefore make sure that IsLoggedIn
can only be set when the browser is convinced that the user meant to log
in or the user is already logged in and wants to stay logged in.

Another potential for abuse is if websites don’t call the logout API
when they should. This could allow them to maintain the privileges tied
to logged in status even after the user logged out.

There are several ways the browser could make sure the IsLoggedIn status
is trustworthy:

* Require websites to use WebAuthn or a password manager (including
  Credential Management) before calling the API.
* Require websites to take the user through a login flow according to
  rules that the browser can check. This would be the escape hatch for
  websites who can’t or don’t want to use WebAuthn or a password manager
  but still want to set the IsLoggedIn bit.
* Show browser UI acquiring user intent when IsLoggedIn is set. Example:
  A prompt.
* Continuously show browser UI indicating an active logged in session on
  the particular website. Example: Some kind of indicator in the URL
  bar.
* Delayed browser UI acquiring user intent to stay logged in, shown some
  time after the IsLoggedIn status was set. Example: Seven days after
  IsLoggedIn was set – “Do you want to stay logged in to news.example?”
* Requiring engagement to maintain logged in status. Example: Require
  user interaction as first party website at least every N days to stay
  logged in. The browser can hide instead of delete the credential token
  past this kind of expiry to allow for quick resurrection of the logged
  in session.

### Credential Tokens

Ideally, a new IsLoggedIn API like this would only work with modern
login credentials. HTTP State Tokens could be such a modern piece.
However, to ensure a smooth path for adoption, we probably want to
support cookies as a legacy option.

Both HTTP State Tokens and cookies would have to be explicitly set up
for authentication purposes to work with IsLoggedIn. In the case of both
of these token types, we could introduce an __auth- prefix as a signal
that both the server and client consider the user to be logged in. Or we
could allow HTTP State Token request and response headers to convey
login status. Note that sending metadata in requests differs from how
cookies work.

The expiry of the token should be picked up as a logout by IsLoggedIn.

Cookies have the capability to span a full registrable domain and thus
log the user in to all subdomains at once. HTTP State Tokens have a
proper connection to origins but can be declared to span the full
registrable domain too. We should probably let the credential token
control the scope of the IsLoggedIn status.

Explicitly logging out should clear all website data for the website,
not just the credential token. The reverse, the user clearing the
credential token (individually or as part of a larger clearing of
website data), should also log them out for the purposes of IsLoggedIn.

### Federated Logins

Some websites allow the user to use an existing account with a federated
login provider to bootstrap a new local user account and subsequently
log in. The IsLoggedIn API needs to support such logins.

First, the federated login provider needs to call the API on its side,
possibly after the user has clicked a “Log in with X” button:

```
navigator.initiateLoggedInFederated(destination: secure origin) –> Promise<void>
```

For the promise to resolve, the user needs to already have the
IsLoggedIn status set for the federated login provider, i.e. the user
needs to be logged in to the provider first.

Then the destination website has to call the API on its side:

```
navigator.setLoggedInFederated(
    loginProvider: secure origin,
    username,
    credentialTokenType,
    optionalParams { }
) –> Promise<void>
```

The promise would only resolve if the `loginProvider` had recently
called `setLoggedInFederated()` for this destination website.

## Challenges and Open Questions

* __Grandfathering__. Some websites may not want to prompt an already
  logged in user or take them through an additional login flow just to
  set the IsLoggedIn status.
* __Expiry limit__. What is a reasonable limit for expiry without
  revisit/re-engagement?
* __Single sign-on__. If the browser supports
  [First Party Sets](https://github.com/krgovind/first-party-sets), it
  may support single sign-on within the first party set, for instance
  with an optional parameter includeFirstPartySet: [secure origin 1,
  secure origin 2]`. The browser would check the integrity of the first
  party set claim and potentially ask the user for their intent to log
  in to multiple websites at once before setting the IsLoggedIn status
  for all of them. The expiry of the login status for the first party
  set would likely be controlled by the expiry of the credential token
  for the single sign-on origin. However, there is not browser agreement
  on how to support First Party Sets in a privacy preserving way (see
  [Issue 6](https://github.com/krgovind/first-party-sets/issues/6) and
  [Issue 7](https://github.com/krgovind/first-party-sets/issues/7)).

## Considered alternatives

[This should include as many alternatives as you can, from high level
architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered, and why you decided
against it.]

### [Alternative 2]

[etc.]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated
positions on this work. If you can, list them here with links to
evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for
their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge
them in an ongoing way! It helps build community and, as we only get by
through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in
alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
