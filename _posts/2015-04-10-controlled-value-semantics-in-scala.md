---
layout: post
title: Controlled value semantics in Scala
---
I recently discovered
[AutoValue](https://github.com/google/auto/tree/master/value "AutoValue") for
Java, which uses annotation-driven code generation to ease the writing of
classes with value semantics (that is to say, classes without object identity).
In Scala, one can accomplish a similar result with case classes, but these also
come bundled with a selection of other features -- `copy` methods and
pattern-matching support, for instance. In the case that one only seeks
immutable value semantics, there is a need to hide some of this generated
behaviour. The trick I employed was to use a sealed trait, viz.:

{% highlight scala %}
sealed trait Account {
  def id: AccountId
  def email: Email
}

object Account {
  case class Account_ private[Account]
    (val id: AccountId, val email: Email) extends Account

  def create(id: AccountId, email: Email): Account =
    Account_(id, email)
}
{% endhighlight %}

The private case class buys us appropriate `equals` and `hashCode`
implementations, but the trait hides the unwanted `copy` and `apply` methods
from the consumer. The trait's companion object provides the `create` factory
method, meaning that the trait can in effect be treated as a concreate type.

Note: the `toString` method will be slightly off (mentioning the
private `Account_` type as opposed to the public `Account` trait), but this is
of no consequence in my case. Your mileage may vary.
