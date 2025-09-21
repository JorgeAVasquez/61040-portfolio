# Exercise 1: Reading a Concept – GiftRegistration

## 1. Invariants

- **Counts**  
  For every `Request`, `count ≥ 0` — and `count` must equal the number of requested items minus the sum of `count` across its `Purchases`.  
- **Requests vs. Purchases**  
  Every `Purchase` must reference an `Item` for which there is a corresponding `Request` in the same `Registry`.

**More Important:** Invariant 1 is more important because it ensures no over-purchasing (you can’t buy more items than requested).  
**Action Most Affected:** `purchase`.  
- It preserves the invariant by checking that the request exists and has at least `count` available, then decrementing the request’s `count` accordingly.

## 2. Fixing an Action

**Problematic Action:** `removeItem`.  
- **Issue:** If there are existing `Purchases` for that `Item`, removing the `Request` would break the invariant linking purchases to requests.  
- **Fix:** Modify `removeItem` to only allow removal if `count = 0` and no purchases exist, or to archive the request rather than deleting it (keeping the linkage intact).


## 3. Inferring Behavior

**Can a registry be opened and closed repeatedly?**  
Yes — `open` only requires the registry to be inactive, and `close` requires it to be active. Nothing forbids re-opening after closing.  

**Reason to Allow:**  
This is useful if the recipient wants to temporarily make the registry private (e.g., while making changes) and then re-open it later.

## 4. Registry Deletion

Registry deletion might matter for data hygiene, but in practice it is not critical: registries are often kept for history, analytics, or sentimental reasons (especially for weddings). Soft-deletion or archiving would be preferable to permanent deletion.

## 5. Queries

- **Owner Query:** "Show me all purchases made (with purchaser names)."  
- **Giver Query:** "Show me which items are still available (with remaining counts)."

## 6. Hiding Purchases

**Augmentation:**  
Add a flag `hidePurchases: Flag` in each `Registry`.  
- If `hidePurchases = true`, then the `owner` cannot view purchases until the registry is closed.  
- Modify `purchase` action’s effect to still record purchases but make them invisible to owner queries while hidden.

## 7. Generic Types

Using generic types (`User`, `Item`) allows this concept to be reused across different contexts (weddings, birthdays, baby showers) and different item representations (SKU codes, database IDs).  
- Representing items by name/description/price would make the concept brittle and tied to one catalog format.  
- Generics keep the concept abstract and implementation-agnostic.

# Exercise 2: Extending a Familiar Concept

## 1 & 2. Concept State & Actions Requires/Effects

```text
concept PasswordAuthentication
purpose limit access to known users
principle after a user registers with a username and a password,
  they can authenticate with that same username and password
  and be treated each time as the same user
state
  a set of Users with
    username: String
    passwordHash: String
actions
  register(username: String, password: String): (user: User, token: String)
    requires no existing User has this username
    effects
      create a new User with
        username = username
        passwordHash = hash(password)
        confirmed = false
        secretToken = generateToken()
      return the created user and secretToken

  authenticate(username: String, password: String): (user: User)
    requires a User exists with
      username = username
      passwordHash = hash(password)
      confirmed = true
    effects
      return that User
```
## 3. Essential Invariant
**Invariant:**
- Each username must uniquely identify a single user.

**Preservation:**
- register checks that no user exists with the same username before creating a new one, preserving uniqueness.  
- authenticate never mutates state, so it cannot break the invariant.
## 4. Email Confirmation
```
state additions:
    Users include
      confirmed: Flag
      secretToken: String?
actions addition:
  confirm(username: String, token: String)
    requires a User exists with username = username,
      confirmed = false, and secretToken = token
    effects
      set confirmed = true
      clear secretToken
```
register() now generates a secretToken and returns it along with the User.
The application layer sends this token to the user via email.
confirm must be called with the correct token to activate the account.
authenticate only works if confirmed = true.

# Exercise 3: Comparing Concepts – Passwords vs. Personal Access Tokens
## PersonalAccessToken Concept Specification
```concept PersonalAccessToken
purpose provide secure, revocable, and script-friendly authentication for a user
principle after a user generates a token,
  they can use the token (with their username) to authenticate,
  until the token expires or is revoked by the user
state
  a set of Tokens with
    owner: User
    value: String
    scope: PermissionSet
    expirationDate: Date?
    active: Flag
actions
  createToken(owner: User, scope: PermissionSet, expirationDate: Date?): (token: Token)
    requires owner exists
    effects
      generate a random value
      create a new token with
        owner = owner
        value = generated value
        scope = scope
        expirationDate = expirationDate
        active = true
      return the created token

  revokeToken(token: Token)
    requires token exists and active = true
    effects
      set active = false

  authenticate(username: String, tokenValue: String): (user: User)
    requires a Token exists with
      owner.username = username
      value = tokenValue
      active = true
      (and either no expirationDate or expirationDate > now)
    effects
      return token.owner
```
## Differences from PasswordAuthentication

**Purpose:**
- PasswordAuthentication is for interactive human login where the user remembers a password.
- PersonalAccessToken is for programmatic or automated authentication — e.g., CLI tools, scripts, or integrations — where storing a password in plaintext would be risky.

**State & Mutability:**
- Passwords are tied one-to-one with users and rarely change.
- Tokens can be created, revoked, and scoped multiple times per user — they are intentionally short-lived and disposable.

**Security Model:**
- Tokens are long, random, and system-generated (no human-memorable requirements).
- Tokens can carry scopes (fine-grained permissions), unlike passwords which usually grant full access.
- Tokens can expire or be revoked independently, reducing risk if leaked.

**User Experience:**
- Passwords must be remembered and typed.
- Tokens must be stored securely (e.g., in a secret manager), but are not meant to be memorized.

## Suggested Improvements to GitHub Documentation

GitHub’s current docs saying to “treat your access tokens like passwords,” is true for security, but confusing conceptually. It obscures their revocability, scoping, and intended use cases.
I would improve the documentation by:
- Explicitly contrasting tokens with passwords:
  - “Unlike passwords, tokens are generated by GitHub, can be limited in scope, and can be revoked or allowed to expire at any time.”
- Adding a small comparison table with when to use passwords vs. tokens (human login vs. automation).
- Highlighting best practices for token storage (environment variables, secret managers) rather than just saying to "treat [them] like passwords."

This would help users understand why tokens exist, not just that they are “like passwords.”

# Exercise 4: Defining Familiar Concepts
## Concept 1: URL Shortener
```concept URLShortener
purpose map long URLs to short, user-friendly links
principle when a user submits a long URL,
  the system generates (or accepts) a short unique suffix
  which maps to the original URL and can be used for redirection
state
  a set of Links with
    longURL: String
    shortSuffix: String
    creator: User?
actions
  createLink(longURL: String, shortSuffix: String?): (link: Link)
    requires no existing Link has this shortSuffix (if provided)
    effects
      if shortSuffix is null, generate a random unused suffix
      create a new Link with longURL, shortSuffix, and optional creator
      return the created Link

  lookup(shortSuffix: String): (longURL: String)
    requires a Link exists with this shortSuffix
    effects
      return the associated longURL

  deleteLink(shortSuffix: String)
    requires a Link exists with this shortSuffix
    effects
      remove the Link from the set
```
**Notes:**
- Supports both user-defined and auto-generated suffixes by allowing an optional shortSuffix parameter.
- Allows link deletion to support expiration or cleanup.
- Does not specify analytics (e.g., click tracking) since that’s outside the core mapping functionality.

## Concept 2: Billable Hours Tracking
```
concept BillableHours
purpose record time spent by employees on projects for billing purposes
principle an employee starts a session on a project with a description,
  and ends the session when finished,
  producing a record that can be billed
state
  a set of Sessions with
    project: Project
    employee: User
    description: String
    startTime: DateTime
    endTime: DateTime?
actions
  startSession(project: Project, employee: User, description: String): (session: Session)
    requires no active session currently exists for this employee
    effects
      create a new Session with given project, employee, description
      startTime = current time
      endTime = null
      return the session

  endSession(session: Session)
    requires session exists and endTime = null
    effects
      set endTime = current time

  autoEndInactiveSessions(timeout: Duration)
    effects
      for each session with endTime = null and startTime older than timeout,
        set endTime = startTime + timeout
```
**Notes:**
- Handles forgotten sessions by introducing autoEndInactiveSessions, which sets a default cutoff (ex., 8 hours after start).
- Guarantees one active session per employee to avoid overlapping billing periods.
- Leaves billing rate calculations to another concept or downstream process.

## Concept 3: Time-Based One-Time Password (TOTP)
```
concept TimeBasedOTP
purpose strengthen authentication by requiring a one-time time-based token
principle a user registers a shared secret with the server,
  then periodically generates time-based codes on a device,
  which must be provided alongside username/password to authenticate
state
  a set of OTPSecrets with
    owner: User
    secretKey: String
actions
  registerSecret(owner: User, secretKey: String)
    requires owner exists and has no existing secretKey
    effects
      create a new OTPSecret for this owner with secretKey

  generateCode(secretKey: String, timestamp: DateTime): (code: String)
    effects
      return a numeric code derived from secretKey and timestamp (not stored)

  verifyCode(owner: User, code: String, timestamp: DateTime): (success: Boolean)
    requires owner exists and has a secretKey
    effects
      return true if code matches the expected result for secretKey and timestamp
```
**Notes:**
- Improves security by mitigating risks of password theft (ex., database leaks or phishing) since an attacker would also need the user’s TOTP device.
- Vulnerable to device theft (if phone is stolen and unlocked) and real-time man-in-the-middle attacks (if code is intercepted and used quickly).
- Does not model cryptographic details (ex., HMAC-SHA1, time step length), focusing only on the concept structure.
