---
title: "Pointing Multiple Domains to Github Hosted Sites With Cloudflare Workers"
date: 2019-03-06T09:40:24-08:00
draft: true
description: A simple, Workers-based approach to corral my multiple domains to display my website.
---

Buying `.dev` domains was recently all the rage. When I searched for `gabbi.dev`, however, I saw that `gabbi.fish`
was also available. All my handles are @gabbifish, so this domain was too good to be true. And, for $30, `gabbi.fish` was a 
steal compared to some of the crazy dollars folks spent on .dev domains!

Of course the first thing I want to do is point my shiny new domain to my preexisting website. 
I already serve my static website on `gabbifish.com` using GitHub web hosting. This requires gabbifish.com 
to have a DNS CNAME record that points to my GitHub-provided subdomain, `gabbifish.github.io`. Additionally, I must share my 
domain name, `gabbifish.com`, with GitHub in a CNAME text file in my GitHub pages repo. 

From what I can deduce, GitHub checks the `Host` header in HTTP
requests to its hosting IP addresses, checks the `Host` value to see if it matches a user-provided domain,
and uses this `Host` header value to map a request to a particular website hosted on GitHub.

This setup works just dandy for one domain. Requests to `gabbifish.com` were upstreamed to GitHub servers,
which responded with my website's static content. Pointing multiple domains to GitHub servers, however, is [not natively supported](https://help.github.com/en/articles/troubleshooting-custom-domains#multiple-domains-in-cname-file). 
In fact, GitHub only allows you to map one domain to a GitHub hosted site! 

Unsurprisingly, setting up a DNS CNAME record from gabbi.fish to gabbifish.github.io failed; GitHub only upstreams requests with the header `Host: gabbifish.com`. Clearly I needed another redirection strategy.

One approach I [saw online](http://blog.teracy.com/2013/08/08/multiple-github-custom-domains/) is to set up an nginx server.
This nginx instance would rewrite certain requests that match regexes. Assume that A.com is CNAME-d to a.github.io. If the 
owner of `A.com` wants to have their domain `B.com` proxied to `a.github.io,` they use nginx to rewrite requests to `B.com` such that 
they are upstreamed to `A.com`. GitHub hosting accepts a request from `A.com` and sends the expected response, which nginx proxies back to the requester of `B.com`. In short, nginx is used to modify the HTTP host header before
proxying the request to the domain that is actually registered with GitHub.

I could spin up an EC2 instance and run nginx in it, but 

1. This sounds a tedious (though it would be fun exercise in basic nginx config),
2. Reconfiguring my DNS records to point to my nginx server would be inconvenient,
3. It's 2019 and there is certainly some *aaS that will make this task easier, and  
4. The latest episode of the Bachelor is online and this is apparently THE episode where Colton finally jumps the fence. I want to set up my new domain quickly so I can watch this.

![lmao](https://media.giphy.com/media/35KmTMq0pcdJzC7ED5/giphy.gif "lmao")

I wanted to use the simplest possible service to achieve this goal. I've gained exposure to Cloudflare Workers at work 
(disclaimer: I work at Cloudflare) and thought that forwarding a request via simple JavaScript logic would be much faster than spinning up nginx and performing the whole nginx song and dance. Functions as a service are super easy to use: you can plug in code with no server overhead, and deploy it in a snap. 

Here was the simple script I used to forward requests from gabbi.fish to gabbifish.com and return the expected
content from gabbifish.com.

```js
/* 
 * Listens to events on my Cloudflare Worker route, gabbi.fish/*
 */
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

/**
 * Take the request to gabbi.fish and rewrite the request Host to gabbifish.com.
 * Then fetch the content at gabbifish.com and return it.
 * @param {Request} request
 */
async function handleRequest(request) {
  let path = request.url.replace("https://gabbi.fish", "https://gabbifish.com")
  const response = await fetch(path)
  return response
}
```

My worker route, `gabbi.fish/*`, is a wildcard that ensures that all requests to gabbi.fish are forwarded to gabbifish.com.
This ensures that all subresource requests (e.g. for images) are properly loaded as well.

So, this isn't true forward proxying--in this case, the request is not being modified, but used to 
create an entirely new request. In the Web Workers environment, `Host` is a [forbidden header](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name), meaning I can't
specify it or modify it.

Still, this approach has served me very well and is simple to follow. It's stateless and much easier to maintain than an nginx instance. 

Perhaps the true moral of the story is that I should host my website on Netlify (which supports multiple domains CNAME-ing to a hosted site). ¯\\_(ツ)_/¯
