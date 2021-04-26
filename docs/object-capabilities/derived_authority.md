---
title: "Derived authority"
section: "Derived authority"
menu:
  toc:
    parent: "object-capabilities"
    weight: 10
toc: true
---

## [Plan]

* demonstrate basic use in combination with file system access etc.
* show how a derived authority can be constructed
* refer to the patterns for more complex scenarios

## [TODO]

* [ ] come up with a better title for the section
* [ ] toc stuff and linking

## Prelude [TODO remove h2]

[TODO what is meant by ambient authority?]

Here is a simple program that connects to example.com via TCP on port 80 and quits:

```pony
use "net"

class MyTCPConnectionNotify is TCPConnectionNotify
  let _out: OutStream

  new iso create(out: OutStream) =>
    _out = out

  fun ref connected(conn: TCPConnection ref) =>
    _out.print("connected")
    conn.close()

  fun ref connect_failed(conn: TCPConnection ref) =>
    _out.print("connect_failed")

actor Connect
  new create(out: OutStream, auth: AmbientAuth) =>
    TCPConnection(auth, MyTCPConnectionNotify(out), "example.com", "80")

actor Main
  new create(env: Env) =>
    try Connect(env.out, env.root as AmbientAuth) end
```

* the main actor is passed the ambient authority as env.root. This is an unforgeable instance
of the AmbientAuth primitive: no code outside the glue that creates the main actor can create
it.
* more specifically, the `env.root` field is of type `AmbientAuth | None`, so the actor must `try`
to convert it by calling `env.root as AmbientAuth` [maybe discuss by None is possible here?]
* the main actor then creates a `Connect` actor and passes the authority on. This second actor
uses it to create a TCPConnection. It is here that the authorization is effectively checked:

```
TCPConnection(auth, MyTCPConnectionNotify(out), "example.com", "80")
```

The TCPConnection requires an authority as first parameter, and since the compiler checks that
the correct type was passed, this guarantees that a TCPConnection can only be created by an
actor holding the needed authorization. If you look into the implementation of the `TCPConnection`
constructor, you will notice that the authorization is not even used beyond the declaration of the
parameter.

## Delegate authorization

Imagine we don't trust the `Connect` actor, so we don't want to provide it with more authority
than needed. For example, there is no point in granting it filesystem access.

The `TCPConnection` constructor's first parameter is of the type
```
type TCPConnectionAuth is (AmbientAuth | NetAuth | TCPAuth | TCPConnectAuth)
```
which looks like a nice hierarchy of authority, in order of decreasing privilege. We can pass
a reduced authority to our `Connect` actor like this:

```
actor Connect
  new create(out: OutStream, auth: TCPConnectionAuth) =>
    TCPConnection(auth, MyTCPConnectionNotify(out), "example.com", "80")

actor Main
  new create(env: Env) =>
    try Connect(env.out, TCPConnectAuth(env.root as AmbientAuth)) end
```

and we can be sure it cannot access the filesystem or listen on a TCP or UDP port. This is possible
because there is a TCPConnectAuth constructor which accepts the AmbientAuth.

You can see this pattern over and over in the standard library.

## Authorization-friendly interface

Consider the above example again, but this time let's treat the `Connect` actor as coming from
a 3rd party package. If the author of that package had not considered which authority exactly
was necessary, it would not be possible to restrict the authoritization given to its code.

[Is this true?]
[What if I need Tcp + File access (but not UDP)? (demonstrate how to accomplish that)]
