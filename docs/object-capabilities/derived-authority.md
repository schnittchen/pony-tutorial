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

Recall the definition of capability from [ref]:

> A capability is an unforgeable token that (a) designates an object and (b) gives the program the authority to perform a specific set of actions on that object.

What gives our program the authority to access the network or the file system? It's the operating system that created the process running our program. We call this *ambient authority*.
The operating system does not understand that our program is divided into objects which might require only a part of this authority.

The ambient authority is represented as the `AmbientAuthority` object passed to the main actor as `env.root`. Instead of allowing unfettered access to the outside world,
Pony reifies the ambient authority and requires you to handle it explicitly.

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

By passing the ambient authority to the main actor, the glue code that creates it
authorizes it to do certain things.
We will see shortly how this authorization is required when interacting with the outside world.
The `AmbientAuth` constructor is private, so that the instance can only be created by the glue
itself.

More specifically, the `env.root` field is of type `AmbientAuth | None`, so the actor must `try`
to convert it by calling `env.root as AmbientAuth` [maybe discuss by None is possible here?]

The main actor then creates a `Connect` actor and passes the authority on. This second actor
uses it to create a TCPConnection. It is here that the authorization is effectively checked:

```
TCPConnection(auth, MyTCPConnectionNotify(out), "example.com", "80")
```

The TCPConnection requires an authority as first parameter, and since the compiler checks that
the correct type was passed, this guarantees that a TCPConnection can only be created by an
actor holding the required authorization. If you look into the implementation of the `TCPConnection`
constructor, you will notice that the authorization is not even used beyond the declaration of the
parameter.

## Delegate authorization

In order to handle our own code and that of others more safely, and also to understand our code better,
we want to split the authority up, and only grant the particular authority a piece of code actually
requires.

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
