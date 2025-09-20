# Problem Set 2: Composing Concepts

## Table of Contents:

- [Exercise 1](#exercise-1-concepts-for-url-shortening)
- [Exercise 2](#exercise-2-synchronizations-for-url-shortening)
- [Exercise 3](#exercise-3-extending-the-design)

## Exercise 1: Concepts for URL Shortening

### 1. Contexts

Context defines the scope within which generated nonces must be unique. The context corresponds to the `shortUrlBase` in a URL shortening service. This allows a short suffix to be reused across different bases without worrying about conflict. Thus, the context is a namespace that localizes nonce uniqueness to a specific base URL.

### 2. Storing used Strings

The specification chooses to represent the state as a set of used strings, describing the behavior abstractly: each generated nonce must be unique and the set of strings records which strings have already been produced to ensure this property.

In the counter-based implementation, we keep a counter for each context, incrmeenting whenever `generate` is called.

The set of used strings in the specification corresponds to the first i strings produced by the counter's encoding scheme where i is the current value of the counter.
When we add a new string, this corresponds with incrementing the counter by 1 in the counter implementation.

### 3. Words as Nonces

Using common dictionary words as nonces makes shortened URLs easy to remember and share, which is especially useful for situations like distributing a link for a class activity or survey. However, a drawback is that the randomly chosen word will likely have no relation to the actual content of the link, leading to confusion or misleading users.

Currently, the `NonceGeneration` concept generates arbitrary strings that need to be unique within a context. Thus, I'd modify the `generate` action. Instead of producing any arbitrary string and returning it if it's not been used in the context, the `NonceGeneration` concept should have an additional restriction in its effects clause: each `generate` returns a string not returned before for that context and it must be in a provided dictionary of common words.

## Exercise 2: Synchronizations for URL Shortening

### 1. Partial Matching

In the first sync, the goal is to trigger nonce generation when a request to shorten a URL is made. The `NonceGeneration.generate` action only needs the short URL base as its context, so only it apears as argument in the `when` clause.

The second sync takes place after the nonce has been generated. At this point, we have the `targetUrl`, `shortUrlBase`, and the newly generated `nonce`, all of which is required to create a mapping between the short URl and target URL. Thus, both `shortUrlBase` and `targetUrl` must appear in the `when` clause of the second sync.

### 2. Omitting Names

The convention isn't used in every case because sometimes explicitly naming variables improves clarity. When the argument name and the bound variable are different, or when multiple actions in a sync share similarly named variables, explicitly labeling them makes it clear which values are being passed where. Without explicit names, the specification can become confusing or ambiguous, especially in more complex synchronizations with multiple inputs and outputs.

### 3. Inclusion of Request

The request action is included in the first two syncs because it triggers the process of generating and registering a URL. In contrast, the third sync occurs after the short URL has already been created. At that point, the URL exists and can have an expiry set. It wouldn't make sense for an ongoing `Request.shortenUrl` call to trigger `setExpiry` because that would imply it's setting an expiration on a URL not yet shortened. Hence, the request action doesn't appear in the third sync.

### 4. Fixed Domain

Currently, alternative domain names are implemented by allowing the input of a `shortUrlBase`. If the application always used a fixed domain, the synchronizations would change as follows:

- The `Request.shortenUrl` action would no longer need to take shortUrlBase as a parameter; it would default to the fixed domain.
- The `NonceGeneration.generate` action would no longer need a context argument since all links share the same domain and therefore the same namespace for uniqueness.
- The `UrlShortening.register` action and any outputs would also no longer need `shortUrlBase` as a parameter; the fixed domain could be harcoded wherever needed.

### 5. Adding a Sync

Below is the new sync for what to do when resources expire.

- **sync** delete
- **when** `ExpiringResource.expireResource(): (resource: Resource)`
- **then** `UrlShortening.delete(shortUrl: resource)`

## Exercise 3: Extending the Design

### 1. Additional Concepts

Below are concepts created to extend the design of URL shortening services.

- **concept** Analytics \[User\]
- **purpose** tracks analytics for a specified short URL
- **principle** a user creates a shortened URL; the user visits the shortened URL several times; the user attempts to view its analytic; the user sees the analytics tracked
- **state**
  - a set of Analytics with
    - a sourceUrl String
    - a counter Counter
    - an owner User
  - a set of Counters with
    - a count Number
- **actions**
  - `createAnalytics(resource: String, owner: User): (analytics: Analytics)`
    - requires: resource exists and its analytics hasn't been initialized
    - effects: creates an analytics object with a counter initialized to 0
  - `incrementCount(analytics: Analytics)`
    - requires: analytics exists
    - effects: increments the counter associated with the given analytics object
  - `viewAnalytics(resource: String, user: User): (analytics: Analytics)`
    - requires: resource has an associated analytics and the owner matches that of the analytics
    - effects: returns current state of the analytics

### 2. Additional Syncs

Below are the synchronizations that must be created.

#### Sync 1: Analytics Creation

- **sync** analytics creation
- **when**
  - `Request.shortenUrl(targetUrl: String, user: User)`
  - `UrlShortening.register(shortUrlSuffix: nonce, shortUrlBase, targetUrl): (shortUrl: String)`
- **then** `Analytics.createAnalytics(resource: shortUrl, wner: User)`

#### Sync 2: URL Lookup

- **sync** url lookup
- **when** `UrlShortening.lookup(shortUrl: String): (targetUrl: String)`
- **then** `Analytics.incrementCount(resource: shortUrl)`

#### Sync 3: View Analytics

- **sync** view analytics
- **when** `Request.viewAnalytics(shortUrl: String, user: User)`
- **then** `Analytics.viewAnalytics(source: shortUrl, user: User): (analytics: Analytics)`

### 3. Assessing Modularity

#### _Allowing users to choose their own short URLs_

To allow users to choose custom short URLs, instead of having a `NonceGeneration` concept, a `NameGeneration` concept would be needed instead. The concept would operate as follows.

- **concept** NameGeneration \[Context\]
- **purpose** allows users to generate a unique name for shortened URLs
- **principle** a user registers to create a shortened URL by providing a URL base, target, and cusom name; if the name is available, the short URL is created; the short URL can be clicked to access the original content of the URL it shortened
- **state**
  - a set of NameGenerations with
    - a set of Contexts
  - a set of Contexts with
    - a set of name Strings
- **actions**
  - `generateName(context: Context, name: String): (name: String)`
    - requires: context exists and the name is not already used in that context
    - effects: adds the name to the context
  - `deleteName(context: Context, name: String)`
    - requires: context exists and name exists in the context
    - effects: removes the name from the context

Additionally, the `URLShortening` concept would need to modify its synchronizations:

##### Generate:

- **sync** generate
- **when** `Request.shortenUrl(shortUrlBase, name)`
- **then** `NameGeneration.generate(context: shortUrlBase, name: String)`

##### Register:

- **sync** register
- **when**
  - `Request.shortenUrl(targetUrl, shortUrlBase, name)`
  - `NameGeneration.generate(context: shortUrlBase, name: String): (name: String)`
- **then** `UrlShortening.register(shortUrlSuffix: name, shortUrlBase, targetUrl)`

#### _Using the 'word as nonce' strategy to generate more memorable short URLs_

Instead of generating a random string in the nonce generation, only words that exist in a common dictionary would be allowed. No other modification will be needed, and the Nonce generation concept can remain as a Nonce generation concept since the current version of `NonceGeneration` uses the set of strings as an abstract function, mapping its size to a counter. Explicitly, the change to `NonceGeneration` will be as follows on the `generate` action:

- `generate (context: Context) : (nonce: String)`
  - effect: returns a string not already used by this context and that exists in a provided dictionary of common words

#### _Including the target URL in analytics_

This feature is undesirable. Analytics are intended to track clicks on individual shortened URLs, which may have different owners. Aggregating across target URLs would violate owner-specific privacy and conflate counts from different users.

#### _Generate short URLs that are not easily guessed_

This feature is undesirable in the context of typical shortening services. Sohort URLs are meant to be easily shared, and the underlying resources are usually publicly accessible. Making URLs harder to guess would hinder usability without providing significant security benefits.

#### _Supporting reporting of analytics to creators of short URLs who have not registered as user_

To implement this feature, all references to the 'User' field from the `Analytics` concept (and any associated syncs or actions) would need to be removed. Additionally, requests would no longer need to pass in information about the user. Note that accesses to analytics remains per shortened URL but lacks ownership restrictions, sacrifing privacy and user-specific access control. Depending on the intended policy, it may be undesirable.
