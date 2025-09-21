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

# Extending the Design
