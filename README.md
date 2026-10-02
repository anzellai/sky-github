# sky-github

A Sky Lang package for **GitHub OAuth sign-in + user profile**. Three
modules, ~250 lines of Sky, no third-party SDKs. Apache 2.0.

## What this is (and what it isn't)

This package covers the "sign in with GitHub" flow end-to-end:

- Build the OAuth authorize URL.
- Exchange the `code` returned at the callback for an access token.
- Fetch the authenticated user's profile.
- Check org membership for allowlist gates.

It deliberately does **not** include:

- GitHub App / installation-token / RS256 JWT minting — that's
  a separate `sky-github-app` library when there's a real consumer.
- Repository / branch / commit / pull-request APIs — likewise.
- Session minting, token storage, cookie management — the caller
  owns those (the example shows one shape).

## Install

```bash
sky add github.com/anzellai/sky-github
```

Then in your Sky source:

```elm
import Github.Oauth as Oauth
import Github.User as GhUser
import Github.Types exposing (AccessToken, User)
```

## MVP API

### `Github.Oauth`

```elm
authorizeUrl :
    { clientId : String
    , redirectUri : String
    , scopes : List String         -- e.g. ["read:user", "user:email"]
    , state : String                 -- CSRF token; caller generates + verifies
    }
    -> String

exchangeCode :
    { clientId : String
    , clientSecret : String
    , code : String
    , redirectUri : String           -- MUST match the authorizeUrl value
    }
    -> Task Error AccessToken
```

### `Github.User`

```elm
current     : AccessToken -> Task Error User
orgMembership : AccessToken -> String -> Task Error Bool
```

### `Github.Types`

```elm
type alias AccessToken =
    { token : String, scope : String, tokenType : String }

type alias User =
    { id : Int
    , login : String
    , name : Maybe String        -- Nothing when GitHub returns null or empty
    , email : Maybe String        -- Nothing when user marks email private
    , avatarUrl : String
    , htmlUrl : String
    }

emptyToken : AccessToken
emptyUser  : User
```

## Quick start

```elm
import Sky.Core.Prelude exposing (..)
import Sky.Core.Crypto as Crypto
import Sky.Core.Task as Task
import Sky.Http.Server as Server
import Github.Oauth as Oauth
import Github.User as GhUser


-- Step 1: kick off the redirect.
handleSignin : Request -> Task Error Response
handleSignin req =
    Crypto.randomToken 16
        |> Task.andThen
               (\state ->
                   let
                       url =
                           Oauth.authorizeUrl
                               { clientId = clientId
                               , redirectUri = redirectUri
                               , scopes = [ "read:user", "user:email" ]
                               , state = state
                               }
                   in
                       Task.succeed
                           (Server.redirect url
                               |> Server.addCookie (Server.cookie "oauth_state" state)))


-- Step 2: callback handler — verify state, exchange code, fetch user.
handleCallback : Request -> Task Error Response
handleCallback req =
    let
        code = Server.queryParam "code" req
        returnedState = Server.queryParam "state" req
        cookieState = Server.getCookie "oauth_state" req
    in
        if returnedState /= cookieState then
            Task.succeed (Server.text "csrf mismatch" |> Server.withStatus 400)

        else
            case code of

                Nothing ->
                    Task.succeed (Server.text "missing code" |> Server.withStatus 400)

                Just c ->
                    Oauth.exchangeCode
                        { clientId = clientId
                        , clientSecret = clientSecret
                        , code = c
                        , redirectUri = redirectUri
                        }
                        |> Task.andThen
                               (\token ->
                                   GhUser.current token
                                       |> Task.map (\user -> userToResponse user))
```

See `examples/01-signin-flow/` for the full working app.

## Env var conventions

This package reads NO env vars on its own — every credential is
passed as a function argument. Consumers typically wire env vars
at the call site:

| Var | Used for |
|---|---|
| `GITHUB_CLIENT_ID` | OAuth app client id (public) |
| `GITHUB_CLIENT_SECRET` | OAuth app client secret (server-only, never to browser) |
| `OAUTH_REDIRECT_URI` | Where GitHub posts the callback (must match GitHub OAuth app config) |
| `OAUTH_STATE_SECRET` | (optional) HMAC key for state-cookie signing |

## Security notes

- **State / CSRF.** The caller is responsible for generating a
  fresh random `state` per sign-in attempt, stashing it in a
  one-shot `HttpOnly; Secure; SameSite=Lax` cookie, and verifying
  it matches on the callback **before** calling `exchangeCode`.
  Without this check, an attacker can trick a victim into signing
  in as the attacker's account ("login CSRF"). The bundled
  example uses `Crypto.randomToken 16` for the state.
- **Tokens are secrets.** Treat `AccessToken.token` as a bearer
  credential — never log it, never put it in a URL, store
  at-rest only after appropriate encryption / key management.
  The package itself never stores tokens.
- **Minimal scopes.** Identity-only sign-in needs
  `["read:user", "user:email"]`. Don't request scopes beyond what
  the app actually uses.
- **Org-membership gating returns False on 404.** GitHub returns
  404 both for non-members AND when the token doesn't carry
  `read:org` scope; for an allowlist check, those are
  indistinguishable (and the safe default is to reject access).
  If you need to disambiguate, inspect `accessToken.scope`.
- **`client_secret` never reaches the browser.** All HTTP calls
  in this package are server-to-server.

## Layout

```
sky-github/
├── sky.toml                  — package manifest (lib exports)
├── LICENSE                   — Apache 2.0
├── src/
│   ├── Main.sky              — stub (sky build needs an entry)
│   └── Github/
│       ├── Types.sky         — AccessToken + User records
│       ├── Oauth.sky         — authorizeUrl + exchangeCode
│       └── User.sky          — current + orgMembership
├── tests/
│   └── GithubTest.sky        — URL-builder + JSON-decoder smoke
└── examples/
    └── 01-signin-flow/       — Sky.Http.Server app demonstrating
        ├── sky.toml          —   the full /signin → /callback →
        └── src/Main.sky      —   /profile → /logout round-trip.
```

## Verification

```bash
# Build the library in isolation.
sky build src/Main.sky

# Run the test suite (22 assertions, no network).
sky test tests/GithubTest.sky

# Build + smoke-test the example (prints the authorize URL, then
# listens on :8001).  No real OAuth credentials needed — the
# config-missing pages render at /signin without them.
cd examples/01-signin-flow
sky build src/Main.sky
GITHUB_CLIENT_ID=Iv1.test_placeholder \
GITHUB_CLIENT_SECRET=test_secret \
./sky-out/app
```

## Status

Pre-1.0. The MVP API surface above is the v0.1 contract.
Breaking changes will be flagged in commit messages. Real
v1.0 ships when there are two production consumers exercising
the same shape (sky-lang.org admin + SkyDeploy migration are
the two we're targeting).

## License

Apache License 2.0. See `LICENSE`.
