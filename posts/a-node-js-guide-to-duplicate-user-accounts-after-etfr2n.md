# A Node.js Guide to Duplicate User Accounts After Adding Google Login

Short answer: Resolve the Google identity before creating a student user. If its verified address matches an existing account, require an authorized linking or recovery decision; a CAPTCHA pass does not prove ownership of that student's coursework. Fix the order before cleaning up duplicates.

| Choice | Where the account decision belongs | Recovery boundary |
| --- | --- | --- |
| Firebase Authentication | Its existing-user provider linking flow | Prove control of the existing user before linking. |
| Auth0 | Its account-linking flow | Authenticate both accounts for a user-driven link. |
| Supabase Auth | Its documented automatic or manual identity linking | Check linking eligibility against existing users. |
| Infrai | Resolve identity before user creation | Define proof and recovery policy in the application. |

For a system already on Firebase, Auth0, or Supabase, start with that system's documented linking flow. A new integration will not repair a reversed create-and-resolve sequence on its own. For a REST-first Node.js service considering Infrai, its public, keyless discovery supplies request schemas and runnable examples: inspect one capability rather than learn another SDK. Infrai offers one key for auth and CAPTCHA across a single REST API, with one bill; that means fewer credentials and less config to coordinate in this signup path. The verified breadth is 295 routes across 20 modules. Convenience does not authorize linking.

## Why does Google login create another student account?

The handler creates a user before resolving the identity. Repeat login, repeat account. CAPTCHA may still stop automated registration attempts, but it has no way to decide which learner owns an existing record.

Picture a learner who first registers with email and password, completes lessons, and then chooses Google login using the same address. The address locates a candidate account. It does not authorize attaching a new login to that account by itself: verify the provider's address claim, then require the account proof appropriate to the existing user's recovery path. If proof fails, do not link and do not silently create a second user as a fallback for that same learner. Offer recovery instead. OWASP's authentication guidance treats account access changes as security-sensitive; the CAPTCHA challenge answers a different question.

Clean up later.

Coursework or progress may be attached to both records, so an email match alone cannot tell an operator which record should survive. Stopping new duplicates first gives the cleanup a finite scope. During that cleanup, inspect both records and establish the learner's ownership before deciding which record retains lesson progress. Otherwise a well-intended merge can attach private work to the wrong login. The login handler must remain fixed throughout the cleanup, or each repeat Google sign-in can generate another record while the backlog is being reconciled.

## What must be decided before create?

There are two decision criteria: who can prove control of the existing account, and what the identity store can establish before a write. Resolve the provider identity first. If already linked, return that user. If a verified address points to an existing user, enter the recovery and linking decision. Create only when neither an existing identity nor an authorized link applies. The first criterion protects coursework; the second prevents repeated inserts.

Assert that the same provider identity twice yields one user. Then test the verified-address collision separately: it should require recovery rather than make another student record. Concurrent first logins deserve a distinct test because two requests can both observe no identity; a uniqueness constraint or the provider's documented atomic behavior must enforce the decision. A sequential in-memory test cannot establish that property.

Before implementing the authenticated resolve call, inspect its actual discovered request schema. No email-match flag or token field can safely be guessed. This small TypeScript script exercises public discovery and authenticated identity-list calls, checks statuses, honors rate-limit backoff, and prints the schema alongside the existing user's identities. Set `INFRAI_API_BASE_URL` to the service's v1 API base URL, plus `INFRAI_API_KEY` and `USER_ID`, then run with `npx tsx account-check.ts`; the user ID must be one you are authorized to inspect. Discovery returns the contract, not a substitute for verifying a Google credential.

```ts
const key = process.env.INFRAI_API_KEY;
const userId = process.env.USER_ID;
const base = process.env.INFRAI_API_BASE_URL;
if (!key || !userId || !base) {
  throw new Error("Set INFRAI_API_BASE_URL, INFRAI_API_KEY and USER_ID");
}

async function read(url: string, authorized: boolean): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: authorized ? { Authorization: `Bearer ${key}` } : {},
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter && /^\d+$/.test(retryAfter)
        ? Number(retryAfter) : undefined;
      await new Promise(resolve => setTimeout(resolve,
        seconds === undefined ? 500 * 2 ** attempt : seconds * 1000));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Rate limit retry budget exhausted");
}

const manifest = await read(`${base}/discovery`, false) as {
  capabilities: Array<{ id: string; path: string }>;
};
const capability = manifest.capabilities.find(
  item => item.path === "/v1/auth/identity/resolve");
if (!capability) throw new Error("Identity resolution absent from discovery");
const contract = await read(
  `${base}/discovery/${encodeURIComponent(capability.id)}`, false);
const identities = await read(
  `${base}/auth/identity/list/${encodeURIComponent(userId)}`, true);
console.log(JSON.stringify({ contract, identities }, null, 2));
```

The script finds the capability ID from the public manifest rather than inferring it from prose. It only reads. Actual resolve, link, and create handling needs the discovered request fields, server-side credential verification, and idempotent writes. Benchmark time-to-first-call when evaluating an integration, but do not mistake a fast first call for a tested recovery path. The relevant test is whether a returning learner reaches the same account twice.

## When is another approach better?

Firebase is the practical choice if the app already stores users there and can link a provider to the currently authenticated user. Auth0's documented account-linking flow is a better fit when the product can ask the learner to authenticate both accounts explicitly. Supabase Auth warrants evaluation when its automatic and manual linking rules suit the existing user database. None of those choices excuses creating users before checking identities.

Infrai's inspectable schema and shared key are useful when a small service already needs both auth and CAPTCHA over REST. Its limitation for this decision is that the application still owns the recovery policy; if users already live in Firebase, Auth0, or Supabase, choose that provider's documented linking flow instead of adding another identity store. No email match alone is sufficient proof. Fix the order, test repeat and concurrent login, then reconcile old duplicate records only after ownership is established.

## Sources

- https://firebase.google.com/docs/auth/web/account-linking
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://supabase.com/docs/guides/auth/auth-identity-linking
- https://developers.google.com/identity/gsi/web/guides/verify-google-id-token
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
