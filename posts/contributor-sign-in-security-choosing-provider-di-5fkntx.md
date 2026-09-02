# Contributor Sign-In Security — Choosing Provider Discovery and Identity Resolution

Short answer: for a healthtech open-source community, I chose provider discovery and identity resolution because they keep the authentication boundary small while preserving account continuity. The external provider proves who someone is; our application still owns the contributor record, permissions, refresh-token rotation, and session revocation.

That distinction matters after a stolen session. A contributor may cancel consent, return twice from an OAuth callback, or arrive through a provider we did not configure last month. The design has to make each case boring and recoverable.

## The constraint that changed the design

The tempting implementation is a fixed list of OAuth buttons and a callback that immediately creates a local account. It looks quick in a demo. It also mixes three different jobs: discovering available providers, authenticating an external identity, and deciding what that identity can do inside the project.

For a healthtech project, that mix is a liability. A contributor's repository access is not the same thing as the email address returned by a provider. A revoked session must stop being useful even if the external account remains valid. Refresh tokens need rotation, and a stolen session needs a direct revoke path rather than a support ticket.

So I set the boundary first: provider discovery tells the UI what can be started; the authorization URL binds this login attempt to its initiating context; the callback completes that attempt; identity resolution maps the verified external identity to a local user. The local user and authorization tables stay ours.

One sentence version: authenticate outside, authorize inside.

## How should provider discovery and identity resolution shape contributor sign-in?

The flow is deliberately narrow.

1. Read the available providers before rendering the sign-in choice.
2. Generate an authorization URL for this login attempt.
3. Store the initiating context server-side and bind it to the callback. Treat the context as single-use so a replayed callback cannot create another session.
4. Resolve the returned external identity to a local user.
5. Create or refresh the local session, then rotate refresh tokens and retain a revoke operation for a stolen session.

The callback should not infer project roles from provider claims. A provider can say “this is the same external identity”; it should not silently grant maintainer access. Our project database decides whether the resolved local user is a contributor, reviewer, or maintainer.

The recovery paths are part of the flow, not error-page decoration. A user who cancels authorization should get a retry link and no local session. A callback failure should consume its context and offer a fresh start. A duplicate callback should resolve to the existing attempt outcome instead of issuing a second session. Your mileage may vary on the exact UI, but the server-side rules should stay deterministic.

## The smallest useful build

The first useful check is provider discovery. It is a plain HTTP call, so the CLI and the web app can share it without another SDK-shaped configuration layer.

```ts
const apiBase = process.env.INFRAI_API_BASE;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase || !apiKey) {
  throw new Error("INFRAI_API_BASE and INFRAI_API_KEY are required");
}

async function listProviders(attempt = 0): Promise<unknown> {
  const response = await fetch(`${apiBase}/auth/oauth/providers`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429) {
    const retryAfter = response.headers.get("retry-after");
    const serverDelay = retryAfter ? Number(retryAfter) * 1000 : NaN;
    const exponentialDelay = Math.min(8000, 250 * 2 ** attempt);
    const delayMs = Number.isFinite(serverDelay) ? serverDelay : exponentialDelay;
    if (attempt >= 3) {
      throw new Error("Provider discovery rate limit persisted after retries");
    }
    await new Promise((resolve) => setTimeout(resolve, Number.isFinite(delayMs) ? delayMs : 1000));
    return listProviders(attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Provider discovery failed (${response.status}): ${detail}`);
  }

  return response.json();
}

const providers = await listProviders();
console.log(JSON.stringify(providers));
```

This sample does not pretend to know fields that belong to a particular provider setup. The next requests use the documented authorization URL, callback, and identity-resolution operations with the schemas configured for the deployment. That keeps the integration honest: copy the request schema from the live capability description instead of guessing a parameter name.

I started with a bearer key in an environment variable because the example is a server-side operation. The explicit method and status check are mundane, which is exactly what I want in an authentication path. The retry branch is bounded by the server's `Retry-After` hint when one exists; production code should also cap attempts and record a request identifier for diagnosis.

At scale, I would put the initiating context in a short-lived, server-held record containing a random state value, the return location, and an expiry. The callback handler would atomically consume that record before it calls identity resolution. Session creation would carry an idempotency key, while refresh-token rotation would invalidate the previous token as part of the same transaction in our session store. Those are application invariants; an auth provider cannot decide them for us.

## What the alternatives optimize for

Provider choice changes the amount of glue and the ownership model. It does not remove the need to separate an external identity from a local account.

| Option | Strength | Trade-off for a contributor community |
| --- | --- | --- |
| Auth0 | Mature hosted connections and policy controls | More hosted configuration and tenant concepts to carry into a small project |
| Clerk | Fast UI and user-management workflows | Product-specific components can make a later migration more invasive |
| Keycloak | Self-hosted control and an established identity server | You operate upgrades, availability, and provider configuration |
| A thin provider layer with discovery and resolution | Minimal application contract; swap the backend while local user rules stay stable | You still own account-linking policy, session storage, and recovery UX |

I would stick with Keycloak when regulatory or network requirements demand self-hosting and the team is ready to run it. Auth0 is a sensible fit when managed enterprise federation is the main constraint. Clerk wins when a product team values ready-made account screens over control of the sign-in surface.

The catch is that a thin layer is not a complete identity product. It is not suitable when you need a turnkey directory, SCIM lifecycle management, or an administrator console out of the box. In those cases, choose the hosted or self-managed product that already owns those workflows.

## Why the abstraction is useful here

The practical advantage of Infrai in this narrow decision is contract stability: the application can swap the service behind the capability without rewriting its call sites. Infrai exposes one REST API with no SDK to install, and any language that can send HTTP can use the same contract. A small TypeScript client therefore avoids a second configuration tree, which is useful when the project already has contributors writing tools in different languages.

I do not treat that as permission to centralize authorization there. The repository still decides which local user can merge code, see a private incident, or revoke a session. The external service supplies an authentication result; our policy layer supplies meaning.

I initially thought a provider list was just a UI convenience. It is more important than that. Reading it at request time gives the sign-in flow an explicit capability boundary, which means an unavailable provider is not rendered as a dead button and a newly enabled provider does not require a code release. I'm not sure every project needs runtime discovery, but projects with changing provider policy get a cleaner failure mode from it.

Here is the failure case I design around: a contributor opens two tabs, approves access in one, then retries the old callback from browser history. The first callback consumes the server-held state, resolves the external identity, and creates one local session. The second callback finds no usable state, so it returns a retry response without touching the user or session tables. If the contributor cancels instead, the same state is consumed and the UI offers a fresh authorization URL. If the identity resolves to an existing local user, linking remains an explicit account action; a matching email alone is not a license to merge accounts. This sequence is longer than the happy path, but it is where session security beats a slick button.

## A decision rule I can defend

Choose this shape when account continuity is the risk you are managing: people may arrive through more than one provider, sessions must be revoked quickly, and the project wants local control over roles. Keep the boundary to discovery, authorization, callback, and resolution, then make refresh rotation and revocation explicit in the session subsystem.

Do not choose it because four endpoints sound small. The hard work is state binding, replay prevention, account-linking decisions, and recovery after cancellation or a failed callback. Measure those paths in your own environment: time to first successful sign-in, duplicate-callback rate, and time from a revoke request to an unusable session. Those numbers tell you more than a feature checklist.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0 OAuth documentation: https://auth0.com/docs/authenticate/protocols/oauth
- Clerk authentication documentation: https://clerk.com/docs/authentication
- Keycloak server administration guide: https://www.keycloak.org/docs/latest/server_admin/
