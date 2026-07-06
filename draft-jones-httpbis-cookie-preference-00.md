%%%

# The Cookie-Preference HTTP Header Field
#
# Generation tool chain:
#   mmark (https://github.com/miekg/mmark)
#   xml2rfc (http://xml2rfc.ietf.org/)

Title = "The Cookie-Preference HTTP Header Field"
abbrev = "Cookie-Preference"
docName = "draft-jones-httpbis-cookie-preference-00"
ipr= "trust200902"
area = "Internet"
keyword = ["HTTP", "Cookies", "Privacy"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-jones-httpbis-cookie-preference-00"
status = "standard"

[pi]
subcompact = "yes"

[[author]]
initials = "P."
surname = "Jones"
fullname = "Paul Jones"
organization = "Terrapane Corporation"
  [author.address]
  email = "paulej@packetizer.com"
  uri = "https://paulej.com/"
  [author.address.postal]
  street = "5448 Apex Peakway #121"
  city = "Apex"
  region = "North Carolina"
  code = "27502"
  country = "USA"
[[author]]
initials = "M. A."
surname = "Ramalho"
fullname = "Michael A Ramalho"
organization = "AcousticComms Consulting"
  [author.address]
  uri = "https://ramalho.us/"
  email = "mar42@cornell.edu"
  [author.address.postal]
  street = "6310 Watercrest Way Unit 203"
  city = "Lakewood Ranch"
  region = "Florida"
  code = "34202-5122"
  country = "USA"

# Revision History
#   00 - Initial draft

%%%

.# Abstract

This document specifies a new HTTP request header field, "Cookie-Preference",
that enables user agents to communicate the user's preferred cookie disposition
(e.g., accept all, accept essential only, reject all, or ask) to web servers.
By conveying this preference upfront, the header can facilitate a more seamless
browsing experience while respecting user privacy choices and reducing reliance
on per-site consent dialogs.

{mainmatter}

# Introduction

Concerns over Internet privacy have led websites to present users with
mechanisms for expressing cookie [@RFC6265] handling preferences. Although these
mechanisms are intended to respect user choices, the inconsistent and repetitive
nature of per-site consent dialogs across the Internet results in a degraded
browsing experience for users.

In practice, most websites offer a similar set of options: accept all cookies,
reject all cookies, or accept only essential cookies (with some also supporting
more granular controls or "ask me" modes). Given this convergence, a more
efficient approach would allow user agents to signal the user's preferred cookie
disposition directly to servers via a standardized HTTP [@RFC9110] request
header field. This preference could be configured globally by the user or on a
per-origin basis within the user agent.

This document defines such a header field, "Cookie-Preference", to enable this
signaling.

# Conventions Used In This Document

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document are to be
interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when,
they appear in all capitals, as shown here.

# Motivation

The widespread adoption of per-site cookie consent dialogs, while driven by
privacy regulations such as the EU's ePrivacy Directive and GDPR, has created
significant user experience friction. Users frequently encounter repetitive,
variably designed prompts asking for the same basic choices: accept all cookies,
reject all (or non-essential), accept only essential cookies, or manage granular
settings. These dialogs often interrupt browsing flow, lead to decision fatigue,
and result in inconsistent application across sites due to differing
implementations and Consent Management Platforms.

Previous proposals for HTTP, such as the historical "DNT" (Do Not Track) header
[@I-D.mayer-do-not-track] (which focused narrowly on tracking preferences and
saw limited adoption), and more recent mechanisms like Sec-GPC (Global Privacy
Control) [@W3C.WD-gpc-20250116] (which signals opt-out from
data selling/sharing), address specific aspects of online tracking but do not
provide a comprehensive way to express general cookie acceptance preferences.
They lack direct support for common dispositions like "accept essential only"
or "ask" modes that align with today's consent banner patterns.

This document proposes the "Cookie-Preference" request header field to fill this
gap by offering a standardized, machine-readable signal from the user agent to
the origin server. The header conveys the user's configured preference for
cookie handling upfront in HTTP requests. Servers can use this signal to
adapt their behavior by, for example, omitting non-essential Set-Cookie
[@!RFC6265] headers when the preference is "essential" or "none," thereby
potentially avoiding or simplifying consent dialogs while still respecting user
choices. A secondary motivation for this draft is to encourage a standardized
way to convey the user's cookie preference thereby discouraging user agent
specific methods, as such user agent specific differences could create yet
another form of user experience friction.

The mechanism is advisory: servers are encouraged but not required to honor the
preference, preserving compatibility and avoiding breakage for sites where
cookie functionality is essential. User agents retain control over when and
how to send the header (e.g., globally or per-origin), allowing flexibility
similar to existing privacy settings.

# The Cookie-Preference Header Field

The "Cookie-Preference" header field is a request header field that conveys the
user agent's configured preference for how cookies should be handled by the
origin server.

## Syntax

Cookie-Preference is a Structured Header Field [@!RFC9651] whose value **MUST**
be a single Item (Section 3.3 of [@!RFC9651]).

Its ABNF [@!RFC5234] production is:

    Cookie-Preference = sf-item

The bare-item **MUST** be an sf-token (Section 3.3.4 of [@!RFC9651]) whose value
is one of the (case-insensitive) values in (#cookie-table).

{#cookie-table}
| Value     | Meaning                                                                    |
|:----------|:---------------------------------------------------------------------------|
| all       | Accept all cookies, including non-essential ones                           |
| essential | Accept only strictly necessary cookies (e.g., session, auth, security)     |
| none      | Reject all cookies; server **SHOULD NOT** attempt to set cookies           |
| ask       | Request explicit or granular consent before setting non-essential cookies  |
Table: Cookie-Preference Values

Additional token values **MAY** be defined in extensions or future registrations
in an IANA registry.

The Item **MAY** include Parameters (Section 3.1.2 of [@!RFC9651]) for
extensibility. Parameters follow the bare-item, separated by semicolons with
optional whitespace. Unrecognized parameters **MUST** be ignored by recipients.

The following are non-normative examples:

    Cookie-Preference: essential

    Cookie-Preference: all; level="high"

    Cookie-Preference: ask; granularity="category"

(Note: Parameters like "level" or "granularity" are illustrative only; their
semantics would need to be defined if registered or used in practice.)

## Semantics

When present, the "Cookie-Preference" field signals the user's overall
preference for cookie acceptance on requests to the origin.

User agents **SHOULD** send this header on requests where cookie-setting
behavior is relevant (e.g., navigation requests or sub-resource loads that may
involve cookies). User agents **MAY** apply different values on a per-origin
basis (e.g., based on user-configured site-specific settings) or send a global
default.

Origin servers **SHOULD** inspect the parsed value of Cookie-Preference
(per the parsing algorithm in Section 4 of [@!RFC9651]) and adjust their
cookie-setting behavior accordingly (refer to (#server-behavior)).

The preference is advisory: servers are not required to honor it, and user
agents **MUST NOT** rely on servers complying (e.g., they **SHOULD** still apply
client-side cookie blocking if configured). If parsing fails at the server or
the value is unrecognized, the entire field **MUST** be ignored
(per [@!RFC9651] error handling).

## Server Behavior {#server-behavior}

Origin servers that understand the "Cookie-Preference" header field **SHOULD**
take it into account when deciding whether to include Set-Cookie headers in
responses, but they are not required to do so. The preference is advisory only;
servers **MAY** ignore it entirely if honoring it would break site
functionality, violate other policies, or would be problematic for any other
reason.

When honoring the signaled preference:

- For "essential", servers **SHOULD** limit Set-Cookie headers to those cookies
  classified as strictly necessary (per definitions in relevant privacy
  regulations or equivalent guidelines).
- For "none", servers **SHOULD** refrain from setting any cookies.
- For "ask", servers **SHOULD** restrict themselves to essential cookies and use
  other mechanisms to solicit more specific user consent.
- For "all", servers **MAY** set cookies without restriction.

Once a cookie has been successfully set (i.e., the Set-Cookie header was sent
and processed by the user agent), this document does not require or define any
mechanism for revoking or modifying it based on a subsequent Cookie-Preference
value. Cookie lifecycle and management remain governed by [@!RFC6265] (or its
successor) and user agent policies.

This document does not require or define any mechanism for revoking or modifying
cookies set as a result of a previous Cookie-Preference header. Cookie lifecycle
and management remain governed by [@!RFC6265] (or its successor) and user agent
policies.

Servers **SHOULD NOT** infer strong privacy commitments solely from this header
(e.g., it does not substitute for valid legal consent under applicable
regulations).

# IANA Considerations

This document registers the following entry in the "Hypertext Transfer Protocol
(HTTP) Field Name Registry" [@!RFC9110]:

- Field name: Cookie-Preference
- Status: permanent
- Specification document(s): this document (RFC-to-be)
- Comments: This is a request header field that conveys user preferences for
            cookie handling. It uses Structured Field Values [@!RFC9651].

No additional IANA actions are requested at this time.

# Security Considerations

The "Cookie-Preference" header field conveys user-configured preferences about
cookie handling, which inherently involves privacy-sensitive information.
Implementers must carefully consider the implications outlined below.

## Fingerprinting and Linkability

Sending the "Cookie-Preference" header field reveals information about the
user's privacy settings or attitudes toward cookies. Since the set of possible
values is small and discrete ("all", "essential", "none", "ask"), and many user
agents may send the same default value, the header alone provides limited
entropy. However, when combined with other request information (e.g.,
User-Agent, Accept-Language, or other preference headers), it can contribute to
fingerprinting the user agent or linking requests across origins/sessions.
User agents **SHOULD** mitigate this by:

- Applying the same value globally unless the user explicitly configures
  per-origin exceptions.
- Omitting the header on requests where it is not relevant (e.g., cross-origin
  sub-resource requests that do not involve cookie-setting).
- Randomizing or varying non-essential headers when privacy is a concern
  (though this is outside the scope of this document).

Servers **MUST NOT** rely on this header as a reliable indicator of user
identity or linkability across requests without additional context.

## Spoofing and Non-Compliance

The header is sent by the user agent and can be trivially spoofed by clients,
extensions, or proxies. Servers **SHOULD** treat the preference as advisory only
and **MUST NOT** depend on it for security-critical decisions (e.g., assuming
"essential" means no tracking cookies are needed for compliance with law).

Conversely, user agents **MUST NOT** assume servers will honor the preference;
they **SHOULD** continue to apply client-side cookie blocking or restrictions
as configured by the user.

## Interaction with Legal Consent Requirements

This document defines a technical signaling mechanism and does not define,
modify, or substitute for legal requirements related to obtaining valid consent
for cookies or similar state management technologies.

Servers remain fully responsible for complying with applicable laws regarding
cookie consent, notice, and choice. In particular:

- The presence of a "Cookie-Preference" header (or any specific value within it)
  does not, by itself, constitute legally valid consent under frameworks
  that require explicit, informed, and affirmative user action.
- The absence of the header or reception of an unknown value does not
  imply or grant permission to set cookies.

However, when a user agent sends "Cookie-Preference: all" (or equivalent future
values indicating broad acceptance), servers **MAY** interpret this as the
user's expressed preference to accept cookies without further prompting,
especially in jurisdictions or contexts where opt-out signals are sufficient,
or where the user agent has already obtained and recorded consent on behalf of
the user. Servers **SHOULD** honor such signals where feasible by omitting
consent dialogs and setting cookies consistent with the indicated preference,
thereby improving user experience while still respecting applicable legal
obligations.

Likewise, when a user agent sends "Cookie-Preference: essential", servers
**MAY** interpret this as the user's expressed preference to accept only
cookies that are essential for site operation. If additional cookies are
desired, the site **MAY** prompt the user to accept additional cookies.

This mechanism is intended to reduce reliance on repetitive per-site consent
dialogs by providing a standardized, upfront expression of user intent. User
agents and servers are encouraged to align their implementations with evolving
best practices and regulatory guidance regarding machine-readable preference
signals.

## General Guidance

Implementers should consult [@RFC6973] for broader privacy considerations in
protocol design, and Section 17 of [@RFC9110] for general HTTP security guidance.
In particular, any future extensions that add parameters or new values to
"Cookie-Preference" **SHOULD** include their own privacy analysis.

# Acknowledgments

TBD

{backmatter}
