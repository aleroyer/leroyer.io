---
title: "Using HAProxy to act as an HTTP server"
date: 2022-11-13T17:54:00+02:00
draft: false
---

During my work, I stumbled upon some little problems that often require to deploy several component for just a simple task.

For example, we had to validate with our CDN that our endpoint was own by us. Fair enough, let's set a TXT record on our DNS servers! Well, it doesn't always work that way. Those providers required an HTTP validation with a specific URL and content.

# Context

We have two providers requesting an HTTP endpoint with a specific url/content.

The first one, we will call this one Alice:
- `https://origin-domain/alice/index.html`
- String content, `text/html` with this value `ALICE <origin-domain>`

The second one, we will call this one Bob:
- `https://origin-domain/bob/bob-test-object.html`
- String content, `text/html` with a specific file provided by them

# Defining a solution

Bob is the easiest one, it only requires a file to be served, however HAProxy is a proxy by definition, it doesn't know how to serve a file right? ...right?

Well, we will see that later.

Alice is kind of cumbersome as we need to return the `origin-domain` for each domain we want to validate.

A simple solution is to write a micro-service, maybe just nginx with some configuration but we will then setup a new application, deploy it in an environment (Kubernetes for example) and this is a lot of work for just a simple thing!

# Just hack it!

In my company, there is a value that says `Just hack it!`. When you encounter a new problem, the solution sometimes reside in reusing the stuff you got instead of just creating new solutions. In addition to that, we love the KISS (Keep It Simple Stupid) approach.

Let's see how we can manage that with HAProxy

## Frontend configuration

Inside the frontend block of you HAProxy configuration, we need to declare ACL to match specific URI sent by our providers.

Below an example of ACLs matching our case:

{{< highlight shell "style=dracula" >}}
acl is_alice_test_object path /alice/index.html
acl is_bob_test_object path /bob/bob-test-object.html
{{< /highlight >}}

We will also need to declare some backend.

## Backend configuration

That's where HAProxy will surprise you. The software is actually capable of returning HTTP content without the need of a server. You can just declare a backend that will answer to a specific requests (or multiple).

Configuration can become a bit complex if you are trying to do something like a mini-application but it is handy to solve simple problems like ours.

### Alice's case

For example, to answer Alice request you can set this configuration:
{{< highlight shell "style=dracula" >}}
backend be_alice_validation_service
  description Self service backend to perform Alice validation
  mode http
  http-request return status 200 content-type "text/html; charset=utf-8" lf-string "ALICE %[req.hdr(host)]\n"
{{< /highlight >}}

It will return `ALICE <domain>` for any request you send to this backend. Easy!

`http-request return` [1] is an option to manipulate the request object sent by HAProxy, since we have no server declared in this backend and no ACL it will return a `200` status for every request.

`lf-string` let's you define a string while allowing the use of Log Format variables. Thus we can set dynamically the request host. 

### Bob's case

Bob's is as easy as Alice's. First you will need to have the file you want to serve store somewhere. For the example I will copy it in `/etc/haproxy` where my configuration resides.

Now for the configuration part, it is almost the same as Alice's:
{{< highlight shell "style=dracula" >}}
backend be_bob_validation_service
  description Self service backend to perform Bob validation
  mode http
  http-request return status 200 content-type "text/html; charset=utf-8" lf-file /etc/haproxy/files/bob-test-object.html
{{< /highlight >}}

See, same as Alice's! But this time we used `lf-file` instead of `lf-string`.

It acts the same (let's you use the Log Format variables) but this time you will load the content from a file. It can be useful if you have a complex html file with some dynamic part.

## Combining all together

Now you just need to use the ACL declared earlier in your frontend.

A simple version would be:
{{< highlight shell "style=dracula" >}}
frontend http_frontend
  bind *:443 ssl cert /etc/haproxy/ssl/cert.pem
  mode http
  acl is_alice_test_object path /alice/index.html
  acl is_bob_test_object path /bob/bob-test-object.html
  use_backend be_alice_validation_service if is_alice_test_object
  use_backend be_bob_validation_service if is_bob_test_object
{{< /highlight >}}

You can try this yourself!

# Conclusion

With this simple configuration we avoid the need of deploying nginx or any other application that can perform such simple substitutions. We also learn a bit of HAProxy hidden capabilities.

Thanks to HAProxyTech's Github [2], I was able to discover that HAProxy can do HTTP requests by its own. Check the repo in the source section to discover more use cases.

Reading the documentation of HAProxy is also very helpful, you can almost do everything with HAProxy! Haha!

---
# Sources

1. http://docs.haproxy.org/2.6/configuration.html#4.2-http-request%20return
2. https://github.com/haproxytech/ultimate-configs/blob/2.2/haproxy.cfg