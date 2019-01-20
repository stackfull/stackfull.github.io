---
layout: post
title: JPA OneToOne with Shared Primary Key
date: 2016-08-29T17:19:30+00:00
categories: [blog]
tags:
  - java
  - spring
  - data
  - programming
---

JPA/Hibernate and especially spring-data are incredibly useful for getting
ahead without the tedium of coding up persistence. They aren't to everyone's
taste (some devs hate to lose the explicit data mapping code) but for the right
problems, I wouldn't want to code without them. That said, once you leave the
simple examples behind it can get very complex. My most recent struggle was
trying to get `@OneToOne` mappings working with shared primary keys. A few
misunderstandings left me chasing my tail for a while. I must have spent a
couple of days on this, pecking at it whenever I had time, so I figure it's
worth a write up.

#### Background

I'm working on a system that authenticates users in multiple ways and I have a
specific data model in mind &ndash; a `User` can have optional
`LocalCredentials` and optional `ExternalUserDetails` (via google, github,
etc.). This was originally either-or, but the requirements changed to allow for
externally authenticated users to acquire local creds and so gain more
permissions.

<img class="size-full aligncenter" src="/assets/User-Model.png" alt="User Model" width="404" height="289" />

Users have a unique email address, but the PK is a surrogate here to make it
work nice with REST. The optional parts share this PK, so although the object
model has relationships traversable from the `User` to the optional objects,
the data model relationships are based on the `user_id` and nothing is added to
the `user` table. In the future, other external auth systems will be added, so
this is important.

<img class="aligncenter" src="/assets/User-Data-Model.png" alt="User Data Model" width="390" height="146" />

#### Object Design

The working stack is spring boot using spring-data and spring-security with a
few goodies like lombok to simplify the code. The main point to note about
using spring-data is that we don't use the DAO pattern to manage tables, we
expect to be able to save `User` entities via a `UserRepository` and have any
related (owned) structures updated/persisted automatically. In a DAO system,
with more explicit mapping of the entities, we would always take care to update
`User` entities before persisting `LocalCredentials` with a copy of the user
ID. In the repository pattern, we link the two objects in the normal (Java) way
and expect the mapping layer to handle the rest.

To realise the design in code, we make a bidirectional reference between the
`User` and `LocalCredentials` and maintain the relation in the setter in `User`
(the 'owner'). The trick to using a shared PK is to add `@Id` to the target
field. Using lombok to simplify, it looks something like this:

```java
@Entity
@Data
@NoArgsConstructor
public class User implements Serializable {

  private @Id @GeneratedValue Long id;
  private @Version Long version;
  private String name;
  private String email;
  @OneToOne(mappedBy="user", cascade=ALL)
  private LocalCredentials local;

  public void setLocal(LocalCredentials local) {
    this.local = local;
    if (local != null) {
      local.setUser(this);
    }
  }

  // ... more to override in the constructor etc.
}
```

```java
@Entity
@Data
@NoArgsConstructor
public class LocalCredentials implements Serializable {

  private String password;
  @OneToOne @Id private User user;
  @Version private Long version;

}
```

This looks like a working object model, but when it comes time to persist
&ndash; the following sample doesn't work:

```java
User user = User.builder().name("Billy Stackfull").email("bill@stackfull.com").build();
userRepository.save(user);
// ...
User user = userRepository.fetchOne(id);
user.setLocal(new LocalCredentials("my-password"));
userRepository.save(user);
// ^ fails once transaction is committed.
```

#### The Problem with Hibernate

This took a stupidly long time to debug. Step debugging through the hibernate
code is painful as there is so much indirection (necessarily) and you have to
take into account all the proxies, listeners, combinations of possible states,
sessions and so on. The symptom presented by the code above is a constraint
violation because the `local_credentials` insert is attempted with a `null`
`user_id`. It's fairly obvious that this means the relationship is being
ignored or broken. In fact the save operation actually breaks the link in the
Java objects! But this is only visible in the `LocalCredentials -> User`
direction, so it's harder to spot as this direction is rarely used.

#### The Solution

Information on the web for this was spotty. There are also numerous ways of
linking objects like this, some long deprecated and some very
Hibernate-specific. I eventually stumbled on the cause by taking [this example
code](https://hellokoding.com/jpa-one-to-one-shared-primary-key-relationship-mapping-example-with-spring-boot-maven-and-mysql/)
and tweaking it to be more and more like my use case until it failed.

Seasoned Hibernaters may have spotted the issue already, but it all comes down
to the use of fields instead of annotated getters. If you de-lombok the entity
definition getters (at least, those on `@Id` fields) and apply the annotations
to the getters, all is well.

#### The Problem with Me

<small>aka PEBKAC</small>

At the point I first discovered this, my reaction was that this must be a bug.
There are so many ways to approach Hibernate that it's possible for edge-cases
to get overlooked. But right now, I'm not so sure. By putting
relationship-maintaining code in the setter, I'm ensuring that direct field
access is not going to work where this relationship is concerned. So using
field access was telling hibernate that it can bypass my own constraints.

But on the other hand, I was definitely not expecting Hibernate to be
re-constructing the relationship from one side only and I'm still not convinced
it should be. _(update: this was accepted as a bug
[HHH-11068](https://hibernate.atlassian.net/browse/HHH-11068))_

What I'm left with is the feeling that this sort of experience is one of the
main reasons there are so many detractors of ORM libraries. I need to be on my
guard that this doesn't unduly influence me next time I have to pick a
persistence stack. It should have some influence, for sure, but developers have
a tendency to be once-bitten-twice-shy.

Having said all that though, this seemed like a very straight-forward use case
and I'm not keen to go back to churning out boiler-plate for a living. Why
can't the magic just work?

