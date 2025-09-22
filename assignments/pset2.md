# Concept Questions

## 1. Contexts  
In the `NonceGeneration` concept, contexts are used to verify uniqueness so two different bases can reuse the same suffix without conflict.  
In the URL shortening app, the context will be the short URL base (i.e. `tinyurl.com` or `mycustomdomain.org`).  
This way, two users using different bases can both generate the nonce `abc123` without colliding, because each base is its own context.  

## 2. Storing Used Strings  
Nonce generation must store previously-used strings to guarantee uniqueness. Without this, the generator might return the same nonce twice.  

If we implement `NonceGeneration` with a simple counter per context:  

- **Implementation state:** a mapping from `context` → `counter` 
- **Specification state:** a set of used strings per context 

The abstraction function maps the counter value `k` to the set of all generated strings `{ "n0", ..., "n(k-1)" }`, allowing the counter 
to represent which strings have been used without explicitly storing them.  

## 3. Words as Nonces  
One advantage comes from the user perspective, as it's easier to read, type, and remember.
A disadvantage is that, since there's a limited supply of dictionary words, there is a higher chance for collision and users may guess
links by trying random words, resulting in lower security.

**Modification:**  
- Add a dictionary of allowed words to the state:  

```
state
  a set of Contexts with
    used: set of Strings
    availableWords: set of Strings
```

# Synchronization Questions

## 1. Partial Matching
In the first sync (`generate`), only `shortUrlBase` appears in the `when` clause because the generation of the nonce (the random suffix) does not depend on the `targetUrl`. At the time of nonce generation, we just need to know which base (domain) we are generating for, not what the link will point to.  
On the other hand, the second sync (`register`) needs both `shortUrlBase` and `targetUrl`, because registering the shortening actually associates the generated nonce (short URL suffix) with its corresponding target URL.

## 2. Omitting Names
The convention of omitting names is only used when it is completely clear what variable is being bound. If we always omitted names, it would become harder to read synchronizations with multiple arguments or return values, since the reader might confuse which variable is being passed where.  
Including explicit names is especially helpful when:
- There are multiple arguments with similar meanings.
- The same variable is used in both arguments and results (risk of ambiguity).
- We want to make the specification self-documenting for maintainability.

## 3. Inclusion of Request
The `Request.shortenUrl` action is included in the first two syncs because both nonce generation and registration are conceptually responses to a user request. They represent part of the end-to-end handling of the user's shorten-URL request.  
In contrast, `setExpiry` does not depend on user input; it is an internal system action that happens automatically whenever a URL is registered. Hence, it does not need to include the request in its `when` clause.

## 4. Fixed Domain
If the application used a fixed domain (e.g., `bit.ly`), we could remove `shortUrlBase` as a variable entirely. The synchronizations would then look like:

```
sync generate
when Request.shortenUrl (targetUrl)
then NonceGeneration.generate (context: "bit.ly")

sync register
when
  Request.shortenUrl (targetUrl)
  NonceGeneration.generate (): (nonce)
then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl)

sync setExpiry
when UrlShortening.register (): (shortUrl)
then ExpiringResource.setExpiry (resource: shortUrl, seconds: 3600)
```
## 5. Adding a Sync
```
sync expireShortUrl
when ExpiringResource.expireResource(): (resource)
then UrlShortening.delete (shortUrl: resource)
```

# Extending the Design

## New Concepts

### Concept: Analytics  
**Purpose:** Track the number of accesses (lookups) for each short URL.  
**Principle:** Each access increments the counter for that short URL.  
**State:**  
- A set of `AnalyticsRecords` with  
  - `shortUrl: String`  
  - `accessCount: Number`  

**Actions:**  
- `createAnalyticsRecord(shortUrl: String)`  
  - Effect: Creates a new record with `accessCount = 0`.  
- `incrementAccessCount(shortUrl: String)`  
  - Requires: A record exists for `shortUrl`.  
  - Effect: Increments the access count by 1.  
- `getAccessCount(shortUrl: String): (count: Number)`  
  - Requires: A record exists for `shortUrl`.  
  - Effect: Returns the number of times the short URL has been accessed.


### Concept: AnalyticsAccessControl  
**Purpose:** Ensure analytics data is visible only to the creator of the short URL.  
**Principle:** Only the user who registered the short URL can view its analytics.  
**State:**  
- A set of `AccessBindings` with  
  - `shortUrl: String`  
  - `owner: User`  

**Actions:**  
- `bindOwner(shortUrl: String, owner: User)`  
  - Effect: Associates a user with a short URL for analytics access control.  
- `authorizeView(shortUrl: String, requester: User): (granted: Boolean)`  
  - Effect: Returns `true` if `requester` is the owner of `shortUrl`, else `false`.


## New Synchronizations

### 1. Sync: Create Analytics Record  
When a new shortening is created, also create an analytics record and bind it to the creator.

```
sync createAnalytics
when
  UrlShortening.register (): (shortUrl)
  Request.shortenUrl (user)
then
  Analytics.createAnalyticsRecord (shortUrl)
  AnalyticsAccessControl.bindOwner (shortUrl, owner: user)
```

### 2. Sync: Track Access
Whenever a shortening is looked up, increment its access count.

```
sync trackAccess
when UrlShortening.lookup (shortUrl)
then Analytics.incrementAccessCount (shortUrl)
```

### 3. Sync: View Analytics
When a user requests analytics for a short URL, check authorization first.

```
sync viewAnalytics
when
  Request.getAnalytics (shortUrl, requester)
  AnalyticsAccessControl.authorizeView (shortUrl, requester): (granted)
then
  if granted then Analytics.getAccessCount (shortUrl)
  else throw UnauthorizedAccessError
```

## Modularity Assessment

### Allowing Users to Choose Their Own Short URLs
**Realization:**  

- Modify the `UrlShortening.register` action to accept an optional `shortUrlSuffix` provided by the user.  
- Precondition: Ensure no existing short URL already uses that suffix to prevent collisions.  
- This can be implemented without changing other concepts. Only the `register` action and the `register` sync need adjustment.  <br>

**Benefit:** Gives users more control and flexibility, allowing them to create meaningful or branded links that are easier to remember and share.  <br>
**Modularity:** High. The feature can be added without impacting analytics or nonce generation.


### Using the “Word as Nonce” Strategy
**Realization:**  
- Extend `NonceGeneration` to include a mode that selects nonces from a dictionary of common words.  
- Maintain the `used set of Strings` to ensure no word is reused in the same context.  <br>

**Benefit:** Easier to remember URLs but may increase risk of collisions due to limited word space.  <br>
**Modularity:** High. This only affects `NonceGeneration`. Other concepts (i.e. `UrlShortening`, `Analytics`) remain unchanged.  <br>



### Including the Target URL in Analytics
**Realization:**  
- Extend `AnalyticsRecord` to store `targetUrl` along with `accessCount`.  
- Modify `incrementAccessCount` to optionally group accesses by `targetUrl`.  <br>

**Benefit:** Allows aggregation of different short URLs that point to the same destination.  <br>
**Modularity:** Medium. Minor extension to `Analytics` but no changes to `UrlShortening` or access control are needed.<br>



### Generate Short URLs That Are Not Easily Guessed
**Realization:**  
- Adjust `NonceGeneration.generate` to produce longer, high-entropy, alphanumeric strings.  
- Ensure uniqueness within the context as usual.  <br>

**Benefit:** Increases security by making short URLs hard to predict.  <br>
**Modularity:** High. This affects only nonce generation. Existing concepts and syncs do not need to change.<br>



### Supporting Reporting of Analytics to Creators of Short URLs Who Have Not Registered as User
**Argument Against:** 
- Analytics are intended to be private and visible only to the registered owner of a short URL.  
- Allowing unregistered users to view analytics breaks access control and could leak sensitive usage data.  
- Without user registration, there is no reliable way to authenticate the “creator” of a short URL.   

This feature is undesirable for privacy and security reasons.

