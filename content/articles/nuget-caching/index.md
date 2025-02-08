---
title: "A solution to gitlab package registry 429's errors"
date: 2025-02-07T17:23:00+02:00
draft: false
mermaid: true
summary: 
tags: 
- dotnet
- caching
- nuget
- package
- registry
- gitlab
- ci
- nginx
- haproxy
---

The GitLab CI ecosystem is full of features and integrations that make it a powerful tool for building, testing, and deploying your applications. It comes with an integrated package registry to share and manage your packages. However, the GitLab SaaS API (gitlab.com) comes with rate-limits which can be a problem for scaling CI/CD pipelines, and quite frustrating when you hit them.  

In this article, I will share a solution I find odd yet very effective: implementing a caching proxy in front of the NuGet GitLab package registry.  

This article isn't a tutorial, but it will provide all required information and gotchas if you want to set up your own caching proxy over the GitLab package registry to improve your CI/CD experience.

# The problem

## The gitlab.com API rate limits

[All requests to the gitlab.com are rate-limited](https://docs.gitlab.com/ee/user/gitlab_com/index.html#gitlabcom-specific-rate-limits). The rate limits are different for authenticated and unauthenticated requests, and depending on the scope. 

In the context of the package registry, the rate limits are as follows:
- Unauthenticated requests: 500 requests per minute (per IP)
- Authenticated requests: 2000 requests per minute (per user)

This isn't that much, even for a medium-sized team. Backend, frontend, QA, deployments... all of these will count towards the rate limit until you hit the limit and get in return a 429 HTTP status code, which will cause your job and pipeline to fail, resulting in a bad and frustrating developer experience and a waste of time.

```
/usr/share/dotnet/sdk/8.0.404/NuGet.targets(174,5): error : Response status code does not indicate success: 429 (Too Many Requests).
```

How to solve this? As specified in the [gitlab.com handbook, no bypass is allowed](https://handbook.gitlab.com/handbook/engineering/infrastructure/rate-limiting/#bypasses), so you can't ask for a rate limit increase. 
How can you scale your CI/CD pipelines without hitting the rate limits?

And if our package registry is private and the token used is the `CI_JOB_TOKEN`, the rate-limit that would apply would be 2000 requests per minute per project, no?

Unfortunately, gitlab.com provides no logs and no metrics, and you'll see that what happens behind the scenes is quite surprising.

## Unauthorized requests to the package registry

The first finding we found out thanks to the debug logs the proxy (we'll talk about it later) is that the dotnet NuGet client will always start with an unauthenticated request to the package registry, even if the NuGet source is configured with a token. The first call to the registry is to get subsequent endpoints, which does not require authentication. The second and third calls are to get the metadata (package versions) and download the package, which require authentication.

This results in a 401 Unauthorized response from the registry (which includes a [www-authenticate header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/WWW-Authenticate) in the response), and the client will then retry with the token.  

{{<mermaid>}}
sequenceDiagram
    Runner->>Registry: /index.json
    activate Registry
    Registry-->>Runner: ✅ List of endpoints
    deactivate Registry

    Runner->>Registry: /metadata/mypackage/index.json
    activate Registry
    Registry--xRunner: 401 Unauthorized ❌</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Registry

    Runner->>Registry: /metadata/mypackage/index.json</br>Authorization: Basic ABCDEFG1234
    activate Registry
    Registry-->>Runner: ✅ List available versions
    deactivate Registry

    Runner->>Registry: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg
    activate Registry
    Registry--xRunner: 401 Unauthorized ❌</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Registry

    Runner->>Registry: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg</br>Authorization: Basic ABCDEFG1234
    activate Registry
    Registry-->>Runner: ✅ mypackage.1.2.3.nupkg
    deactivate Registry
{{</mermaid>}}

This implies two things:
- We have twice as many requests as we actually need (which counts towards the rate limit)
- The first request being unauthenticated, it counts against the unauthenticated rate limit (500 requests per minute). This means that even if you have a token, the first call being unauthenticated, you'd hit the unauthenticated rate limit before the authenticated one. 

That's a bummer... this effectively means all of our projects are limited by the same rate limit, even if they have a different token. Plus the fact that the unauthenticated rate limit is the lowest of all, and that all requests are doubled.  

That also implies we're rate-limited by IP, which can be a problem if you have a lot of runners behind the same public IP. 

## Parallel fetching for NuGet sources

Another finding is that the dotnet NuGet client will fetch the metadata for all sources in parallel. For instance, if your project depends on a public package like `xunit`, the client will fetch the metadata for xunit from nuget.org but also from your private GitLab package registry, even if you don't have any `xunit` package in your registry (it ends up in 404).

This results in a lot of unnecessary requests to the Gitlab package registry, and again, counts towards the rate limit. Given how many `Microsoft.*` and `System.*` packages we usually have in our projects (to name a few), this can be a lot of requests.

{{<mermaid>}}
sequenceDiagram

    participant Runner
    participant Registry
    participant Nuget#46;org

    Runner->>Nuget#46;org: /metadata/xunit/index.json
    activate Nuget#46;org

    Runner->>Registry: /metadata/xunit/index.json
    activate Registry
    Registry--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Registry

    Runner->>Registry: /metadata/xunit/index.json</br>Authorization: Basic ABCDEFG1234
    activate Registry
    Registry-->>Runner: ❌ 404 Not Found
    deactivate Registry

    Nuget#46;org-->>Runner: ✅ List of packages
    deactivate Nuget#46;org

    Runner->>Nuget#46;org: /download/xunit/2.9.3/mypackage.2.9.3.nupkg
    activate Nuget#46;org

    Runner->>Registry: /download/xunit/2.9.3/mypackage.2.9.3.nupkg
    activate Registry
    Registry--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Registry

    Runner->>Registry: /download/mypackage/2.9.3/mypackage.2.9.3.nupkg</br>Authorization: Basic ABCDEFG1234
    activate Registry
    Registry-->>Runner: ❌ 404 Not Found
    deactivate Registry

    Nuget#46;org-->>Runner: ✅ xunit.2.9.3.nupkg
    deactivate Nuget#46;org
{{</mermaid>}}

This issue can be addressed with the help of the [Package Source Mapping](https://learn.microsoft.com/en-us/nuget/consume-packages/package-source-mapping) feature, but it requires manual work and maintenance in each project. That said, it is considered a good practice for security reasons (prevents supply chain attacks), so it's at least worth a look.

# Caching packages with a read-through caching proxy

The most effective solution we found was to implement a proxy in front of the Gitlab package registry, not only to cache packages, but also to handle unauthenticated requests (401) and requests for packages that are not in the registry (404), hence reducing the overall number of requests, but also avoiding the unauthenticated rate limit.

## Choose your weapon

There are many HTTP proxies available out there to achieve this. The requirements are for the proxy to be able to cache, preferably in-memory, be able to defines some rules over headers and paths, and be able to edit headers and response bodies. We chose to go with our own implementation using rust and [hyper.rs](https://hyper.rs/), but proxies like Nginx and HAProxy are also good choices.  

See [the configuration we used for cacheus](https://github.com/ogxd/cacheus/tree/main/examples/nuget_cache).

## Replacing the endpoints

The initial request to the GitLab package registry will return a list of endpoints. Unfortunately, these endpoints are on the gitlab.com domain, so we need to replace the domain with the domain of our proxy for NuGet to continue fetching through our proxy. The logic is simple: if the path is `/api/v4/groups/xxxxx/-/packages/nuget/index.json` (check what yours is), we replace all occurrences of `gitlab.com` with our proxy domain in the response body.

## First call without authorization header

The second rule is to return a 401 to all requests that don't have an `Authorization` header. This will prevent the client from retrying with the token, and will also prevent the request from counting towards the rate limit. A special header `WWW-Authenticate: Basic realm="GitLab Package Registry"` but also be added to the response to make the client understand that it needs to authenticate. This way we are no longer rate-limited by IP, nor to the lowest rate limit, and we divide the number of requests by two.

{{<mermaid>}}
sequenceDiagram
    participant Runner
    participant Proxy
    participant Registry

    Runner->>Proxy: /metadata/mypackage/index.json
    activate Proxy

    Note over Proxy: "Authorization" header is missing

    Proxy--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Proxy

{{</mermaid>}}

## Calls for nuget.org packages on private registry

We chose to return a 404 from the proxy itself for most requests for packages that are not in the registry. We achieved this by looking for the presence of `metadata/microsoft.`, `metadata/system.` or `metadata/xunit` in the request path. This won't cover all cases, but we found out that it would remove about 70% of the requests made to the registry for our main projects.

{{<mermaid>}}
sequenceDiagram
    participant Runner
    participant Proxy
    participant Registry

    Runner->>Proxy: /metadata/xunit/index.json</br>Authorization: Basic ABCDEFG1234
    activate Proxy

    Note over Proxy: Path contains "metadata/xunit"

    Proxy--xRunner: ❌ 404 Not Found
    deactivate Proxy

{{</mermaid>}}

## Caching the metadata and packages

Finally, we can cache the response of the remaining calls. The metadata shall not be cached for too long since it can change (eg you just released a new package), but the packages can be cached for much longer since they are immutable for a given version. We chose to cache the metadata for 1 minute and the packages for 1 day. This is all done in-memory with a simple LRU cache. Some developers were concerned by the potential memory usage of the cache, but we found out it wouldn't get above 300mb for 500 packages, which is very reasonable.

## Https / .NET 9

One last but important detail: dotnet 9 made it mandatory to use HTTPS for a package sources. It can be disabled with `--allow-insecure-connections` when adding the NuGet source, but otherwise it's best to have a valid certificate on your proxy (or on your ingress if you use one).

# Results

Here is what our I/O looks like using the proxy, considering the cache is empty (which is the worst case scenario, and quite unlikely once the cache has already been solicited for a few jobs).

{{<mermaid>}}
sequenceDiagram
    participant Runner
    participant Proxy
    participant Registry

    Runner->>Proxy: /metadata/mypackage/index.json
    activate Proxy
    Proxy--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Proxy

    Runner->>Proxy: /metadata/mypackage/index.json</br>Authorization: Basic ABCDEFG1234
    activate Proxy

    Proxy->>Registry: /metadata/mypackage/index.json</br>Authorization: Basic ABCDEFG1234
    activate Registry

    Registry->>Proxy: ✅ List of packages
    deactivate Registry

    Note over Proxy: Cache for 1 minute

    Proxy->>Runner: ✅ List of packages
    deactivate Proxy

    Runner->>Proxy: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg
    activate Proxy
    Proxy--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Proxy

    Runner->>Proxy: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg</br>Authorization: Basic ABCDEFG1234
    activate Proxy

    Proxy->>Registry: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg</br>Authorization: Basic ABCDEFG1234
    activate Registry

    Registry->>Proxy: ✅ mypackage.1.2.3.nupkg
    deactivate Registry

    Note over Proxy: Cache for 1 day

    Proxy->>Runner: ✅ mypackage.1.2.3.nupkg
    deactivate Proxy
{{</mermaid>}}

Here is what it looks like in the best case scenario:

{{<mermaid>}}
sequenceDiagram
    participant Runner
    participant Proxy
    participant Registry

    Runner->>Proxy: /metadata/mypackage/index.json
    activate Proxy
    Proxy--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Proxy

    Runner->>Proxy: /metadata/mypackage/index.json</br>Authorization: Basic ABCDEFG1234
    activate Proxy

    Note over Proxy: Cache hit!

    Proxy->>Runner: ✅ List of packages
    deactivate Proxy

    Runner->>Proxy: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg
    activate Proxy
    Proxy--xRunner: ❌ 401 Unauthorized</br>WWW-Authenticate: Basic realm="GitLab Package Registry"
    deactivate Proxy

    Runner->>Proxy: /download/mypackage/1.2.3/mypackage.1.2.3.nupkg</br>Authorization: Basic ABCDEFG1234
    activate Proxy

    Note over Proxy: Cache hit!

    Proxy->>Runner: ✅ mypackage.1.2.3.nupkg
    deactivate Proxy
{{</mermaid>}}

As you can see, the proxy and cache are able to handle a large portion of the requests without calling the Gitlab package registry. In practice, we observe a reduction of I/Os between 80% and 100%. This combined with the fact that we're now theoretically rate-limited to 2000 requests per minute per project instead of 500 globally, we can now scale our CI/CD pipelines without fearing of hitting the rate limits again 🎉.