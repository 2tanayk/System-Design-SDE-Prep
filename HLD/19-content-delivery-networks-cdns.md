# Content Delivery Networks (CDNs)

## 1. Why do we need a CDN?

A Content Delivery Network (CDN) is a geographically distributed network of servers that delivers cacheable content from locations closer to users.

Imagine an application hosted in Mumbai, with users in:
- Mumbai
- London
- New York
- Singapore

Without a CDN, requests for resources hosted by the application may have to travel all the way to the application's infrastructure in Mumbai. This adds network latency, especially for users far away.

Now consider how many users might request the same resources:
- Images and profile pictures.
- JavaScript and CSS bundles.
- Fonts.
- Videos and other media.
- Public pages or API responses that are safe to share.

It is wasteful for every user to fetch the same static file from the same origin. A CDN keeps copies at distributed edge locations so eligible requests can be served closer to users.

**Core idea:** move copies of cacheable content closer to users, reduce repeated requests to the origin, and improve delivery performance.

A CDN is one of the caching layers in a system:

| Layer | Examples | Typical use |
|---|---|---|
| External application cache | Redis, Memcached | Shared application data across backend servers |
| CDN / edge cache | Cloudflare, Fastly, Akamai, CloudFront | Static media and cacheable HTTP responses close to users |
| Client-side cache | Browser HTTP cache, mobile storage | Avoid repeated network requests from the client |
| In-process cache | Local server memory, Caffeine | Small, frequently read data local to one application instance |

The supplied caching-video notes describe a CDN as the default edge layer for static media at scale. They include an illustrative cross-continent latency comparison of roughly 250–300 ms versus 20–40 ms from an edge location. Treat these as example figures, not guaranteed latency: real results depend on geography, routing, network conditions, content size and provider infrastructure.

## 2. The most important question: how does the CDN get into the request path?

This is the key architectural concept.

A CDN does not automatically intercept requests after they reach the application server. We configure the CDN as the public entry point for the hostnames or traffic paths that we want it to handle.

### Without a CDN

Suppose the application is available at:

    https://www.example.com

DNS resolves the hostname to the application's public infrastructure, such as a load balancer.

    Browser
       |
       | www.example.com
       v
    DNS resolution
       |
       v
    Load balancer
       |
       v
    Application servers
       |
       v
    Database / storage

The browser connects to the application infrastructure directly.

### With a CDN in front

Now we create a CDN distribution and configure:
- The public hostname, for example www.example.com.
- The origin, for example the application's load balancer.
- Cache rules describing which requests can be cached and for how long.
- Routing behavior for requests that should be forwarded to the origin.

Then we change DNS/proxy configuration so requests for www.example.com resolve to, or are proxied through, the CDN network rather than directly to the load balancer.

    Browser
       |
       | www.example.com
       v
    DNS resolution
       |
       | resolves / routes to CDN
       v
    CDN edge location
       |
       | cache miss or request not cacheable
       v
    Origin / load balancer
       |
       v
    Application servers
       |
       v
    Database / storage

The browser still requests www.example.com. The URL does not need to change. The DNS and CDN configuration changes the network destination for that hostname.

Depending on the provider, this can involve a CNAME or an equivalent DNS/proxy configuration. For example, CloudFront provides a distribution hostname and supports mapping an application hostname to that distribution. Cloudflare's proxied DNS records route HTTP/HTTPS traffic through its network rather than returning the origin address directly.

### What if we only want static files to use the CDN?

We can use separate hostnames:

    https://static.example.com/images/profile.jpg
    https://static.example.com/css/main.css
    https://static.example.com/js/main.js

    https://api.example.com/users/123
    https://api.example.com/orders/456

Configure:
- static.example.com to use the CDN.
- api.example.com to resolve directly to the application's load balancer.

The request paths now look like this:

    Browser
       |
       +-------------------------------+
       |                               |
       v                               v
    static.example.com              api.example.com
       |                               |
       v                               v
    CDN edge                       Load balancer
       |                               |
       v                               v
    Object storage / origin         Application servers
                                       |
                                       v
                                    Database

The CDN receives static-file requests, while API requests bypass it.

### Can one hostname use both CDN caching and application APIs?

Yes. We can put the CDN in front of the hostname and configure path-based cache/origin behaviors:

    www.example.com/assets/*  -> CDN cache behavior
    www.example.com/api/*     -> Forward to application origin; caching disabled or carefully controlled

Important distinction: DNS operates at the hostname level, not at the URL-path level. The CDN receives traffic for the hostname, then its request rules decide what to cache and where to forward each path.

This is the missing piece to remember: **DNS/proxy configuration puts the CDN in front; CDN behaviors decide what happens to each request.**

## 3. What are edge locations and the origin?

### Edge location

An edge location, or Point of Presence (PoP), is a location in the CDN network that can receive and serve user requests.

The CDN's routing system directs a user to an appropriate edge location. The selected location is not necessarily the geographically closest physical server; network routing, availability and provider decisions also matter.

### Origin

The origin is the upstream source from which the CDN retrieves the original content when it needs to fetch it.

The origin does not always mean the application server.

| What is being delivered? | Possible origin |
|---|---|
| Dynamic API response | Application infrastructure / load balancer |
| Server-rendered HTML | Web server or application infrastructure |
| Image or video | Object storage, such as an S3 bucket |
| CSS or JavaScript | Static file server or object storage |

For example, a CDN can fetch an image directly from object storage without involving the application server at all.

A common architecture is:

    Browser
       |
       v
    CDN edge
       |
       v
    Object storage

For a dynamic API response, it may instead be:

    Browser
       |
       v
    CDN edge
       |
       v
    Load balancer
       |
       v
    Application server
       |
       v
    Database

## 4. How does a CDN serve a request?

Suppose the user requests:

    https://static.example.com/images/product-123.jpg

The request is routed to a CDN edge location.

The edge checks whether it already has a usable cached response for this request.

### Cache hit

The object exists in the edge cache and is still fresh.

    Browser
       |
       v
    CDN edge
       |
       | Cache HIT
       v
    Return cached image

The CDN serves the image without contacting the origin for this request.

### Cache miss

The edge does not have the object, or the cached object needs revalidation.

    Browser
       |
       v
    CDN edge
       |
       | Cache MISS
       v
    Origin
       |
       | Return image
       v
    CDN edge
       |
       | Store according to cache policy
       v
    Browser receives image

A subsequent eligible request reaching an edge with a valid cached copy can be served from that cache.

The supplied video notes describe this as a read-through-cache-like behavior: the edge fetches from the origin on a miss and caches the result. This is a useful mental model, though a CDN is a specialized HTTP delivery and reverse-proxy infrastructure, not simply Redis deployed geographically.

## 5. What content should be cached?

### Static content — the ideal starting point

- Images and profile pictures.
- JavaScript and CSS bundles.
- Fonts.
- Videos and media segments.

These resources are often identical for many users and do not change with every request.

### Public, cacheable dynamic content

Dynamic content can also be cached when many users can safely share the same response.

For example:

    GET /api/products/123

Response:

    {
      "id": 123,
      "name": "Mechanical Keyboard",
      "price": 4999
    }

If this response is public and its freshness requirements permit caching, it may be a good candidate.

### Personalized content — be careful

Consider:

    GET /api/account/balance

User A may receive a balance of ₹10,000, while User B receives ₹2,000.

A shared CDN cache must not accidentally reuse User A's response for User B.

Personalized responses should generally use private caching or no-store, unless a carefully designed user-specific cache policy is in place. Never assume that authentication alone automatically makes a response safe from shared-cache mistakes.

The important distinction is:

**Can different users safely receive the same cached response for this request?**

If not, do not put the response into a shared cache without explicitly designing and validating the cache key and privacy behavior.

## 6. How does the CDN know how long content is fresh?

The origin can send HTTP response headers that describe caching behavior. A CDN may use these headers together with its own configured cache rules.

The most important header is Cache-Control.

### Example A: Cache a versioned static asset for one year

    HTTP/1.1 200 OK
    Content-Type: image/jpeg
    Cache-Control: public, max-age=31536000, immutable

Meaning:
- public: shared caches are allowed to store the response.
- max-age=31536000: the response is fresh for 31,536,000 seconds (one year).
- immutable: while fresh, the content is not expected to change.

This is appropriate when a changed asset receives a new URL.

### Example B: Different browser and shared-cache lifetimes

    Cache-Control: public, max-age=60, s-maxage=300

Meaning:
- max-age=60: a private/browser cache can consider the response fresh for 60 seconds.
- s-maxage=300: a shared cache, such as a CDN, can consider it fresh for 300 seconds.

For shared caches, s-maxage takes precedence over max-age.

### Three directives that are easy to confuse

| Directive | Meaning |
|---|---|
| no-cache | The response may be stored, but must be revalidated before reuse. |
| no-store | Caches should not store the response. |
| private | The response is intended for private caches, not shared caches. |

Important interview point: **no-cache does not mean do not cache.** It means the cache must validate the stored response before reusing it. no-store is the directive that prevents storage.

## 7. What if content changes before the TTL expires?

Suppose a product image is cached for one hour. An administrator replaces the image after five minutes.

The CDN may continue serving its old cached copy while it is still considered fresh.

Two common approaches solve this.

### Approach 1: Cache invalidation / purge

Tell the CDN to remove a particular cached object. The next request can then fetch the new content from the origin.

Example:

    Purge: /images/product-123.jpg

Some providers also support purging groups of objects using tags or other identifiers.

### Approach 2: Versioned URLs / cache busting

Instead of overwriting the same URL, publish a new URL whenever the content changes.

    /assets/main.v1.js
    /assets/main.v2.js

Or use content hashes:

    /assets/main.8f31a2.js
    /assets/main.c91d45.js

The new URL identifies a new object in the CDN cache. The application can safely assign a long freshness lifetime to immutable, versioned files.

A common strategy:
- HTML entry documents: short freshness lifetime or revalidation.
- Hashed JavaScript/CSS/images: long freshness lifetime with immutable.
- Frequently changing public content: shorter TTL or revalidation.

TTL is a freshness limit, not a promise that the CDN will keep an object in memory for that entire duration. A CDN can evict an object before its freshness lifetime ends to make room for other content.

## 8. Cache keys — what makes two requests the same?

A CDN needs a cache key to identify which cached response belongs to a request.

A simple cache key might be based on:

    https://example.com/products/123

But requests can differ:

    /products?category=books
    /products?category=electronics

These must not accidentally share the same cached response.

Depending on configuration, a cache key can consider:
- Hostname.
- URL path.
- Query-string parameters.
- Selected request headers.
- Cookies, where explicitly configured.

For example, if the response varies by language, the language selection may need to be reflected in the cache key or handled through a separate URL.

### Why does this matter?

Suppose an application personalizes a response based on a cookie, but the CDN cache key ignores that cookie.

The CDN might incorrectly reuse one user's response for another user.

Therefore:

**A cache key must distinguish requests whenever those requests can produce meaningfully different responses.**

At the same time, adding unnecessary query parameters or headers creates too many distinct cache entries and reduces the cache hit ratio.

The exact default cache key and configuration options vary by CDN provider.

## 9. Origin Shield / Tiered Caching

Imagine millions of users request the same video during a major event.

A CDN has many edge locations. Multiple edges may independently miss their local caches and fetch the same object from the origin.

That can create unnecessary origin traffic.

A tiered caching design introduces an intermediate cache layer between edge locations and the origin.

    Users
       |
       +-------------+-------------+
       |             |             |
       v             v             v
     Edge A        Edge B        Edge C
       |             |             |
       +-------------+-------------+
                     |
                     v
             Regional / Shield Cache
                     |
                Cache miss
                     |
                     v
                   Origin

If the intermediate cache already has the object, an edge can retrieve it there rather than contacting the origin.

Benefits:
- Reduces duplicate origin requests.
- Reduces origin bandwidth.
- Helps protect the origin during traffic spikes.
- Improves cache efficiency across multiple edge locations.

Providers use different names and implementations for tiered caching and origin shielding.

## 10. CDN vs. Redis

These are both caching technologies, but they solve different problems.

| Dimension | CDN | Redis |
|---|---|---|
| Primary purpose | Deliver content close to users | Cache application data close to backend services |
| Location | Distributed global edge network | Usually a centralized or regional cache cluster |
| Typical consumers | Browsers and HTTP clients | Backend application services |
| Typical data | Images, CSS, JS, videos, public HTTP responses | Objects, query results, counters, sessions, frequently accessed records |
| Cache miss | Fetch from origin or upstream cache | Application/cache integration fetches from database or upstream |
| Main benefit | Lower geographic latency and origin bandwidth | Lower database load and backend read latency |

Typical request paths:

**CDN**

    Browser
       |
       v
    CDN edge
       |
       v
    Origin / object storage

**Redis**

    Browser
       |
       v
    Backend application
       |
       v
    Redis
       |
       | Cache miss
       v
    PostgreSQL

A system can use both:
- CDN for images, videos, JavaScript and CSS.
- Redis for frequently accessed product data, query results or other backend data.

## 11. Common CDN trade-offs and failure scenarios

When proposing a CDN, proactively consider:

**Stale content:** an old copy may be served until expiry or invalidation.

**Cache misses:** cold edge locations, first requests and evicted objects still require an upstream fetch.

**Incorrect cache keys:** missing a request dimension can cause incorrect responses or expose personalized data.

**Origin overload:** widespread misses can send a large burst of requests to the origin. Tiered caching and request collapsing can help.

**Cache fragmentation:** unnecessary query parameters or headers can create too many cache variants and reduce the hit ratio.

**Configuration complexity:** cache policies, invalidation, URL versioning, origin access and routing rules must be managed.

A CDN is not automatically useful for every endpoint. Introduce it when request volume, geographical distribution and the cacheability of content justify the additional layer.

## 12. Interview example — design media delivery for a global application

Suppose an application allows users to upload and view public images.

A reasonable starting architecture:

    User uploads image
          |
          v
    Application API
          |
          v
    Object storage
          |
          v
    Image URL

    User views image
          |
          v
    CDN edge
          |
          | Cache hit -> return image
          |
          | Cache miss
          v
    Object storage origin
          |
          v
    CDN edge stores and returns image

Design decisions:
1. Store the original media in object storage.
2. Put a CDN in front of the media hostname.
3. Configure the media hostname's DNS to route through the CDN.
4. Cache public images at edge locations.
5. Use versioned URLs or cache invalidation when an image changes.
6. Set appropriate freshness headers.
7. Keep private/user-specific media out of shared caching unless access control is explicitly designed.
8. Monitor cache hit ratio, origin request volume, bandwidth and latency.

Only introduce more advanced mechanisms, such as tiered caching or origin shielding, when traffic patterns or origin load justify them.

## 13. Interview summary

- **CDN:** A geographically distributed edge network for delivering cacheable content closer to users.
- **Edge location / PoP:** A CDN location that receives and serves user requests.
- **Origin:** The upstream source of the content. It can be application infrastructure, a web server or object storage.
- **How it enters the request path:** DNS/proxy configuration points the desired hostname to the CDN. The CDN then serves or forwards requests according to its rules.
- **Cache hit:** The edge has a usable cached response and can serve it without contacting the origin.
- **Cache miss:** The CDN fetches the content from the origin or an upstream cache.
- **Cache-Control:** HTTP response directives that influence storage and freshness.
- **Cache key:** Identifies which requests may safely share a cached response.
- **Invalidation:** Removes cached content before its ordinary freshness lifetime ends.
- **Versioned URLs:** A reliable strategy for changing static assets while allowing long cache lifetimes.
- **Origin Shield / Tiered Caching:** An intermediate cache that reduces duplicate requests to the origin.
- **CDN vs. Redis:** CDN serves content close to users; Redis serves application data close to backend services.

**Remember:** A CDN is not automatically in the request path. Configure DNS/proxy routing so the CDN is the public entry point for the hostname; then configure its cache and origin behaviors.

## References

### Source material used as the lesson's starting point

- User-provided source: [Caching in System Design Interviews](https://www.youtube.com/watch?v=1NngTUYPdpI). The source's CDN coverage focuses on the CDN/edge caching layer, static media, geographic proximity, and the read-through-cache analogy.

### Additional details added to clarify architecture and HTTP behavior

- [Cloudflare — How Cloudflare DNS works](https://developers.cloudflare.com/fundamentals/concepts/how-cloudflare-works/)
- [Cloudflare — Proxy status](https://developers.cloudflare.com/dns/proxy-status/)
- [Amazon CloudFront — Use custom URLs with alternate domain names](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html)
- [MDN — Cache-Control header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [MDN — HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)
- [Cloudflare — Cache keys](https://developers.cloudflare.com/cache/how-to/cache-keys/)
- [Cloudflare — Get started with Cache](https://developers.cloudflare.com/cache/get-started/)
