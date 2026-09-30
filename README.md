# itb-plugin-oauth

OAuth 2.1 vocabulary for the OTB Gherkin language: request a token, exchange an
authorization code, introspect a token, and assert on the error responses.

Dialect only. This plugin deploys nothing and calls no service — every verb is a
plain HTTP exchange with the system under test.

## Why it exists

A token endpoint takes `application/x-www-form-urlencoded`, and the core
language cannot send one:

- the core's POST verbs assign `Content-Type: application/json` *inside* the
  verb, so a `set header` written earlier in the scenario is silently
  overwritten;
- a form body has to be assembled from run-time values — an authorization code,
  a PKCE verifier, a resource identifier — and the core's `{value}` slot accepts
  only a literal, a `$variable` or a number, never a `concat()`.

Both are deliberate in the core, which is shaped for JSON APIs. Neither is
something a test author can work around, which is why these verbs live here.

Note what that means for review: a feature file that sends form bodies through
the core's POST verbs **compiles clean** and sends them labelled as JSON. The
compiler cannot catch it.

## Use it

Declare the dialect in the feature's tags:

```gherkin
@lang:itb-core-en@^2 @dialect:oauth@^1
```

Client authentication is a request header, so it is set the ordinary way. These
verbs write `Content-Type` and `Accept` into the core's own header map and touch
nothing else, so `set header` and `set bearer token from` keep working beside
them.

```gherkin
Background:
  Given AuthorizationServer is the system under test at "https://as.example.org"
  And Client is infrastructure
  And Client is asked for $clientAuth with "HTTP Basic credentials, as 'Basic <base64>'"
  And set header "Authorization" to $clientAuth

Scenario: a client-credentials grant returns a bearer token
  When Client requests a token from AuthorizationServer at "/token" with grant "client_credentials" and scope "patient/*.read"
  Then $response.status should be 200
  And extract "/access_token" from $response.body as $accessToken
  And $accessToken should not be empty

Scenario: bad credentials are refused
  Given set header "Authorization" to "Basic bm90LWEtY2xpZW50"
  When Client requests a token from AuthorizationServer at "/token" with grant "client_credentials"
  Then $response.status should be at least 400
  And $response.body should be the OAuth error "invalid_client"
```

## The verbs

```gherkin
{actor} requests a token from {actor} at {string} with grant {value}( and scope {value})?( for resource {value})?
{actor} exchanges the code {ref} for a token on {actor} at {string}( with verifier {ref})?( and redirect {value})?
{actor} introspects the token {ref} on {actor} at {string} as {var}
{actor} posts form-encoded to {actor} at {string} with:
{ref} should be the OAuth error {string}
{ref} should carry an OAuth error code
```

The optional parameters are omitted from the request entirely when you leave the
slot off, rather than sent empty — so

```gherkin
When Client exchanges the code $freshCode for a token on AuthorizationServer at "/token"
```

sends no `code_verifier` at all, which is what a PKCE negative test needs.

`posts form-encoded … with:` is the escape hatch for what the other verbs
deliberately cannot express: a malformed request, an unsupported grant type, a
parameter the profile does not use. The doc string is sent verbatim.

## Limits

**No URL encoding.** The form bodies are joined with `concat()` exactly as the
values are given. A scope or resource identifier containing `&` or `=` would
corrupt the body. In practice scopes and endpoint URLs are safe, and an
authorization code is base64url by convention, but if you need an encoded value,
encode it yourself and pass it as a literal.

**No token decoding.** Reading the claims out of an issued JWT is a separate
concern; use [`itb-plugin-jwt`](https://github.com/OpenTestBed/itb-plugin-jwt)
alongside this one.

## Who uses it

IHE IUA, SMART on FHIR, UDAP — any profile that delegates authorization to an
authorization server. The IUA suite in `itb-plugin-authoring` is the worked
example.

## Licence

BSD-3-Clause.
