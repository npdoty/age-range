# Age Range API

## Authors

- Nick Doty (CDT)

## Participate
- [Issue tracker](https://github.com/npdoty/age-range/issues)

## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Age Range API](#age-range-api)
  - [Authors](#authors)
  - [Participate](#participate)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [User-Facing Problem](#user-facing-problem)
    - [Goals](#goals)
    - [Non-goals](#non-goals)
  - [Proposed Approach](#proposed-approach)
    - [Requesting age range information](#requesting-age-range-information)
    - [Asking user permission](#asking-user-permission)
    - [Support voluntary age information, entered into browser UI](#support-voluntary-age-information-entered-into-browser-ui)
    - [Allow sites to request custom age ranges](#allow-sites-to-request-custom-age-ranges)
    - [Required attestations](#required-attestations)
    - [Cross-platform practicality](#cross-platform-practicality)
    - [Guarantees of accuracy](#guarantees-of-accuracy)
    - [Consistent responses](#consistent-responses)
  - [Alternatives considered](#alternatives-considered)
    - [Digital Credentials API](#digital-credentials-api)
    - [HTTP Header Prefer: safe](#http-header-prefer-safe)
    - [HTTP Header Age-Range](#http-header-age-range)
    - [Possible Addition: notarization](#possible-addition-notarization)
  - [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
  - [Stakeholder Feedback / Opposition / Support](#stakeholder-feedback--opposition--support)
  - [References \& acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Age assurance mandates are proliferating.
Several jurisdictions
([California](https://leginfo.legislature.ca.gov/faces/billTextClient.xhtml?bill_id=202520260AB1856),
[New York](https://www.nysenate.gov/legislation/bills/2025/S8102))
have active legislation under consideration
that, if passed, would mandate the provision of age range signals to websites.
This proposal describes an API that enables compliance with this form of legislation.

Providing websites with any information about the age of a user
can be in direct tension to the obligation of a user agent
to [limit the user information that is revealed to sites](https://w3ctag.github.io/privacy-principles/#restrict-data-to-necessary-or-aligned).
Where the provision of age range information is mandated by law,
websites have no choice than to block access
until the information has been supplied.

Where such legal mandates exist
this proposal seeks to maintain a high standard of adherence
to [privacy principles](https://w3ctag.github.io/privacy-principles/)
around data sensitivity, minimization, rights, disclosure, accountability,
and other factors.

Age assurance mandates can be accomplished through several different architectural designs.
In comparison with other approaches for controlling access to age-inappropriate content,
voluntary signaling of age ranges provides some significant advantages.
Websites that have age-dependent content can request an age range,
adjusting their experience accordingly.
Only age range information is revealed;
no user identity or other persistent identifier is revealed.
The decision of what to disclose is handled by the device administrator
and the decision of when to disclose is under user control.

## User-Facing Problem

A user that provides age information to a website
trades limited personal information (their approximate age)
for an age-appropriate experience.
A user may also wish to voluntarily request content or functionality be restricted
as if they were a given age.
This can result in a more positive online experience,
albeit at a cost in privacy.

Legal mandates on website operators
can prevent them from providing service to users.
In those cases, an age range API gives users a choice
about whether to provide information
in exchange for access to the site.
Limiting the information about age to a range of ages
allows sites to provide age-appropriate content
without learning more sensitive information,
such as a precise age or birth date.

### Goals

1. Offer limited information about user age to sites, where necessary
1. Give sites the option to limit their exposure to sensitive information by specifying age ranges of interest
1. Be practically deployable across OSs given platform APIs
1. Provide accountability for what age information websites collect

### Non-goals

1. Verifiable age-related claims
1. Technical enforcement of the purpose of age information collection
1. Other content restriction mechanisms: content labeling, biometrics, or credential-based methods

## Proposed Approach

We propose a single Javascript API to request age range information, along with a well-known resource to provide per-origin guarantees and attestations. The following sections will show the use to accomplish the goals above.

### Requesting age range information

Requesting age range information is done with a simple async Javascript API:

```javascript
let [lowerAgeLimit, upperAgeLimit] = await navigator.ageRange();
```

This provides an inclusive age range as provided by the underlying platform, when available. By default, the only age threshold considered is 18 years, making the expected responses for age limits `[0, 17]` and `[18, Infinity]`.

If age range information is not available,
the range `[0, Infinity]` is returned.
This applies when the platform is unable to provide age information,
when the platform lacks age information,
and when the user does not grant permission.
This response might also be returned
if an application changes the age thresholds it sets.

### Asking user permission

Gathering the user's age range should be a [powerful feature](https://w3c.github.io/permissions/#powerful-features). This would require the user to give express permission to the site with the browser's UI to gather an age range. This mechanism is already used to protect the user's privacy from intrusive uses of [Geolocation](https://w3c.github.io/geolocation/).

### Support voluntary age information, entered into browser UI

If the user’s platform does not support age range information, the user or their guardian may want to voluntarily declare an age range to the browser, so that sites may alter or restrict content.

### Allow sites to request custom age ranges

Custom age ranges would be supported with an argument of age thresholds to the call. There may be no more than 3 ages and they must not be sequential. The browser should also restrict repeated calls so that it doesn't reveal more information that this limit about the user.

```javascript
let [lowerAgeLimit, upperAgeLimit] = await navigator.ageRange([13, 18]);
```

The expected responses for age limits in this call are `[0, 12]`, `[13, 17]` and `[18, Infinity]`. However, if the browser only has information that the user is underage, it may return `[0, 17]`.

### Required attestations

To facilitate the API, a declaration, fetched from a well-known location, evaluated by the user agent to confirm acceptable declarations and relevant age-ranges (`.well-known/age-range-declarations.json`).

```json
{
  "sell-or-share": false,
  "child-safety-only": true,
  "relevant-age-thresholds": [13, 18],
  "updated": "2026-01-16",
  "expires": "2026-12-31",
  "origin": "example.com",
}
```

These are promises, from the site operator, about what information they need and use, and what they will use it for. Like any promise, it could be broken, and accountability would most often have to be after the fact. But accountability through data protection authorities, consumer protection agencies and other out-of-band mechanisms is possible once promises are publicly made. Without this requirement, many companies would be happy to request this data, make no promises about it, and use the data for any purposes (including selling it to others) without any possibility of accountability.

When the site requests an age range, the browser will request the well-known resource. Additionally, the browser will request the age from the platform, if available.  This may trigger OS UI to pop-over the browser. If no age thresholds are given, a default value 18 is used. If the thresholds in the well-known file are not a superset of those in the API call or the other fields do not provide an active attestation for the current site, the browser should block returning a result to the site.

### Cross-platform practicality

As of now, there are age range APIs in development in iOS and Android and no equivalent on desktop OSs. While the available APIs are similar, there are subtle differences that must be smoothed over to provide a unified API to the web.

One difference is the time at which age ranges are declared. Android [only permits one age range per app](https://developer.android.com/google/play/age-signals/use-age-signals-api#custom-age-ranges). This means that even though sites can request a non-default age thresholds, the browser will likely be unable to make that request. If the site requests age thresholds unavailable in the current app, then the browser should reduce the provided signal into the thresholds provided by the platform API. This is done by providing age thresholds outside of the platform’s information. For example a request for a 17 year old’s age with thresholds from the web `[4, 17, 21]` on an Android app would default to a request with thresholds `[13, 16, 18]`. The OS would return range `[16, 17]` and the browser would map that to `[4, 20]`. This leaks the true age range from the OS.

This allows a unified API across platforms while providing as much information as is available to the browser.

### Guarantees of accuracy

This API should not be seen as guaranteeing the age of the current user conforms to the age range presented. Users may configure the device to give an age that is not their own. A browser may be shared by users of different ages. The browser could even provide a [0, Infinity] result arbitrarily.

It is a principle of Web API design that [APIs should be designed such that data returned through an API does not assert a fact or make a promise on the user's behalf about the user or their environment](https://www.w3.org/TR/privacy-principles/#principle-no-facts-or-promises) in the interest of user privacy.

However, device administrators interested in applying this functionality might restrict users from freely configuring age range signals, including by limiting the minimum age to what the platform asserts on operating systems where such an API exists.

### Consistent responses

Once a site has obtained an age range,
subsequent calls to the API should return the same value.
Otherwise, a site could learn when the user ages into the next age range,
revealing their birthday to within a time
that is based on how often they visit the site.

For sites that might make more content available
to users in older age ranges,
users might seek to provide updated age information
when they get older.
Browsers are encouraged to give users an interface
to manually update the age range information that sites receive.

Retaining the last answer given to a site is especially important
when sites change the age thresholds they are interested in.
The browser should use the original range in answering subsequent queries,
expanding the range previously returned to fit the requested thresholds.
For example, a 15 year-old user might provide a site with a range of `[13, 16]`.
If that site subsequently sets thresholds at `[10, 15, 18]`,
the response would need to be expanded to `[10, 18]`,
rather than `[15, 18]`.

This state is site-specific and persistent,
so it should be cleared when site data (such as cookies) are cleared by the user.
It should not be cleared when the site requests state removal,
such as through the [`Clear-Site-Data`](https://www.w3.org/TR/clear-site-data/) mechanism.

## Alternatives considered

### Digital Credentials API

The [Digital Credentials API](https://w3c-fedid.github.io/digital-credentials/), when combined with [zero-knowledge proofs](https://datatracker.ietf.org/doc/html/draft-google-cfrg-libzk-01) over properties of digital identity documents or [selective disclosure](https://datatracker.ietf.org/doc/rfc9901/), provides a way to obtain an age range. However, this approach has a few open challenges: it discloses the issuer to the site (and may be linkable to the user's full identity document); it requires possession of a digital ID by all age-gate-bypassing users; and it is still vulnerable to bypass by credential sharing.

### HTTP Header Prefer: safe

The ["safe" HTTP preference](https://datatracker.ietf.org/doc/html/rfc8674) is an indication by a browser that the client doesn't want to be served "objectionable" content, as defined by the server. This mechanism has seen opposition because it may be insufficiently clear that the user is of a given age when it is not sent, in part because it has not seen sufficient adoption. Its binary nature makes it hard to use for age signalling in the current world where many age thresholds are being put into place, sometimes more than one in a given jurisdiction.

### HTTP Header Age-Range

One alternative approach is the definition of an HTTP header specific to signal age ranges, as a [Client Hint](https://www.rfc-editor.org/rfc/rfc8942). This header, `Age-Range` (or perhaps `Sec-CH-Age-Range`) would have values and semantics identical to the proposed Javascript API’s values, and rely on server-side opt-in through an `Accept-CH` header.

Challenges to this approach may include:
* latency: if the header would significantly alter the contents, would require additional roundtrips (via `Critical-CH` or landing pages)
* permissions: if browsers or operating systems use interactive permission prompts, this wouldn't match, might block on loading pages

Advantages:
* applies to contexts beyond JavaScript
* allows for administrator configuration, without requiring limits on browser extension capabilities

Our understanding is that RFC 8942 requires that Client Hints only be deployed when there is an alternative platform mechanism for revealing the data, so a JavaScript approach might still be required anyway. In that case, a header would be duplicative, and may create latency or blocking on permissions.

### Possible Addition: notarization

Presence of text in a well-known file is not a hard guarantee. We could optionally provide signing of these declarations by a known third party (a notary) to provide some additional measure of accountability, or just to prevent mistaken abuse by copying and pasting declaration files without understanding them.

Notarization is potentially useful for three purposes:

1. ensuring the declaration is (relatively) static and universal, with detectable changes
2. identifying a person or organization that can be held accountable through legal process
3. optional vetting/validation ahead of time, or accountability of removal after abuse is disclosed

This could be the same declaration file, but presented as a JSON Web Token (JWT).

This is bureaucratically costly in that it requires browsers to maintain a list of notaries, and understand what minimal policies they have in place. But it could also speed up distribution of these declarations, rather than relying on fetching during a website visit. And a notary could gather some information, like proof that this is a real business with a known identifier that can be held accountable.

## Accessibility, Internationalization, Privacy, and Security Considerations

We expect no significant challenges with accessibility or security. There are no additional user surfaces to ensure are accessible, nor are there additional security challenges due to the addition of this API.

Internationalization encounters challenges while trying to offer access to the appropriate age threshold information across cultural contexts. Different cultures view different ages as important thresholds. Even within the United States 13, 16, and 18 are insufficient– 15 and 21 also are used culturally and legally to represent ages of majority. However, the Android Play Age Signals only allow three age thresholds per app. This makes it impossible to release a single browser that can be useful internationally. Due to this constraint, we specify a flexible API surface which accurately relays the information available on to the website under the constraints of data minification requested by the site. This matches more directly with the iOS Declared Age Range API.

This API is built-for-purpose to disclose enough age information as is needed by sites to design responsibly, while not disclosing any identity information. The well-known resource offers browsers a mechanism to ensure that only one set of age ranges is used per origin and offers policymakers a lever to hold site operators accountable for a declaration of purpose.

One broader concern of this API is that it facilitates the exclusion of children from some online spaces. This has potential significant downsides, reducing the free expression and access to information by children and teens, particularly those already at-risk and whose parents disagree with that expression or access to information. Any system that provides outside constraints on the actions of children may cause this harm. This approach provides weaker guarantees and is as circumventable as other approaches.

## Stakeholder Feedback / Opposition / Support

If you are an implementer or stakeholder and have feedback, please open an issue or PR to render your position.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

* Rick Byers
* Marcos Cáceres
* Yves Lafon
* Lola Odelola
* Martin Thomson
* Benjamin VanderSloot
* Jeffrey Yasskin

Many thanks to the prior work in this space:

* The [Restricted-to-Adults label](https://www.rtalabel.org)
* The ["safe" HTTP Preference](https://datatracker.ietf.org/doc/html/rfc8674)
* The [Declared Age Range](https://developer.apple.com/documentation/declaredagerange)
* The [Play Age Signals](https://developer.android.com/google/play/age-signals/overview)
* The [Privacy Sandbox Developer Enrollment and Attestations](https://github.com/privacysandbox/attestation)
* The [Digital Credentials API](https://w3c-fedid.github.io/digital-credentials/)
