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

# Extending the Design
