# Hello

(about us)

# Intro

We're here to talk about modern software development.

Now, You know what software development is.
I believe most of you are developing software every day.

So what does "modern" mean?
For us it's particular, liberating, and exciting style of development

# FP + T

It's called functional programming with types.

And we'll get on to what that means in a moment.

But we're not talking about being modern because of any particular product or architecture.
We're not modern because we're using React,
or MongoDB, 
or whatever is hot right now.

It's the ideas and techniques from functional programming that matter.
They give software developers a huge helping hand in every-day programming.
Regardless of the environment you find yourself in.

The core ideas have been around for ages.
FP is about programming using functions,
and using functions as values to other function,
recursion,
constructing nested data structures,
and being able to take them apart again.

We especially like functions that produce no side effects.
What's been realised is that that style is a great fit to scaling in parallel
and across large data sets.
If you've heard of map reduce, you'll know what we mean.

# DITL 

Rather than waffle on about this in the abstract,
we thought we'd show you the day in the life of a functional programmer.

We'll go through the kinds of things you do writing software,
but through the eyes of a functional programmer.

We'll see some actual code.
Now this code is in Scala.
We're not here to teach you the language.
You're all smart people, you'll get the general idea.
Feel free to stop and ask questions, but don't worry about the details too much.
We'll give you the code so you can explore it in more detail if you want to. 

# Take away points

There are two things we want to take away from this.

The first is that functional programming is for everyone.
Some of the things we're going to show you are simple.
But they have weird names. 
That's in part because functional programing has origins and influences from maths and logic.
The idea today is that  you'll see the patterns we're showing as evidence that functional programming isn't reserved for maths and logic experts.

The second thing we want you to watch out for is that these techniques make change easier.
And it does that by making your code better modularized and easier to reason about.

# GO

Let's start with the daily stand up.
The team has been building a web service.
It's a JSON based API behind a customer management system.
The API lets you list customers, add customers, update customers.
Simple stuff.

```bash
$ http GET :8080/customers
HTTP/1.1 200 OK
Content-Type: application/json
```
```json
[
    {
        "id": 1,
        "name": "Alice",
        "phone": "+1 555 1234"
    },
    {
        "id": 2,
        "name": "Bob",
        "phone": "+1 555 5678"
    }
]
```

All we have at the moment is customers. And a customer has an id, a name, and a phone number.

We've picked up a new ticket, and that's to add a subscription indicator to a customer.

[show story card]

> As a customer, I can have a subscription, so that I can receive my copy of the monthly company magazine.

Sounds like we're adding a field to a record.
Not too tricky.
We did a little deeper and find out that a customer might have:

- No subscription
- An Active Subscription
- Lapsed Subscription (...if they've not paid their bills.)

We're going to have to show that in JSON,
store it in the database,
write a state machine to manage transitions between the states.

How could we do that? Instantly a functional programmer says:

> This is a job for an....Algebraic Datatype

I did warn you there were some weird phrases.

This is an incredibly simple idea, you'll wonder what the fuss was about.
We're going to separate the representation from the behaviour.

We'll see some Scala syntax now.
We know we need something called a subscription:

```
sealed trait Subscription
```

A trait is like an interface. 
An important part of this is that it is sealed,
which means "hey compiler, you're about to see all the possible kinds of Subscription"

And now we can list them out:

```
sealed trait Subscription
case object Never  extends Subscription
case object Active extends Subscription
case object Lapsed extends Subscription
```

We're saying that Never is a case of Subscription.
As is Active, and Lapsed.
And those are only the kinds of subscription.

We might also expect that some of these subscriptions have additional attributes.
Perhaps we want to know when an active subscription is due to expire:

```
sealed trait Subscription
case object Never extends Subscription
case class  Active(expires: DateTime) extends Subscription
case object Lapsed extends Subscription
```

Which is fine, each case can have different relevant fields.

We'd include a subscription in our representation of a customer:

```
case class Customer(
  name  : String,
  phone : String,
  sub   : Subscription
)
```

What do we need to do next?
Well, we could look at the database, or the JSON representation,
but we're going to talk about them later.

So let's instead look at a state machine for managing subscriptions.

```
def transition(sub: Subscription): Subscription = ???
```

We want to maybe handle expired subscriptions in some way.
Because this is an ADT we don't have to think much about it.
We have to handle each case in turn:

```
def transition(sub: Subscription, now: DateTime): Subscription =
  sub match {
    case Lapsed | Never  => sub 
    case Active(until)   => ???
  }
```

What we're saying here is if the subscription matches Lapsed or Never,
we have nothing to do and we return the subscription.

In the other case, we want to change the subscription:

```
def transition(sub: Subscription, now: DateTime): Subscription =
  sub match {
    case Active(until)   => if (until > now) Lapsed else sub
    case Lapsed | Never  => sub 
  }
```

So we have a function that we can run with a subscription at some time
and we get back a subscription.
Pure function, no side effects here.
We have separated the description from the behaviour.

This is okay, but ... we probably want to do more here.
Really, we probably want to _do_ things.
Like maybe send a nagging email if the subscriber has lapsed.
Or send an incentive if the customer is expiring soon.
But we don't want side effects.

So what do we do? 

It's another ADT!

```
sealed trait Action
case class EncourageRenewal(c: Customer) extends Action
case class Expire(c: Customer) extends Action
case class Nag(c: Customer) extends Action
case object NoAction extends Action

def transition(customer: Customer, now: DateTime): Action =
  customer.sub match {
    case Active(expiry) if expiry - now < 1.month => EncourageRenewal(customer)
    case Active(expiry) if expiry > now => Expire(customer)
    case Active(_) => NoAction
    case Lapsed => Nag(customer)
    case Never =>  Nag(customer)
  }
```

We still have a pure function.
We have a function from a customer at a particular date,
to an action.  
We don't actually carry out the action, such as sending an email.
We've represented the action.
We can test this function as much as we want,
across any scenario we want,
and we don't have to worry about mocking out email servers
or databases.
This is making our lives as developers easier.

When it comes to actually sending the email, 
we can isolate that.  Moving the damage out to the edges.

What would that look like:

```
def perform(action: Action): Unit = ???
```

Unit here means there was no result.
It signals a side effect is about to happen.

Can anyone tell me how to fill out the body
of this method?

As a clue... ask: what kinds of actions do we have?

The trick here is that we repeat what we've done before.
We've defined our data structure, our ADT, and now we take it apart again:

```
def perform(action: Action): Unit =
  action match {
    case EncourageRenewal(customer) => sendEmail()
    case Nag(customer) => textThemAtMidnight()
    ...
    }
```

We do a bit of thinking to represent the "domain",
the match the heck out of it.

There are good reasons why we like doing this.
One is that the compiler can tell us if we screw up.

What if there's a new kind of Subscription.  A Freebie subscription we give out to special customers.
How do we know we've correctly handled this kind of subscription.
The answer is that, because of that sealed world, the compile will tell us:

[error message here]

Another reason we like ADTs is their versatility.
Has anyone heard of `Option` or `Optional` in Java?

```
val perhapsWeHaveANumber: Option[Int] = ???
```

It's a way to express that we might, or might not, have a value.
It's an ADT.
There are two cases: Some and None.
We can match on that:

```
perhapsWeHaveANumber match {
  case Some(x) => x + 1
  case None => None
}
```

Or how about a List?

```
val numbers = List(1,2,3,4,5)

numbers match {
  case Nil => "No numbers"
  case x :: rest => s"The first number is $x"
}
``

Again, two cases: the empty list, and then a value and another list
which happens to be expressed as `::` in Scala.

# Coffee

So that wasn't so bad. First ticket done by now.
Time for some coffee.

We saw ADTs
Separating a representation from doing it
Helps isolate side effects
Common pattern used over and over.
Compile tells you if you've missed a case.
...makes change easier
...super useful in day-to-day development.


# Back to the issue tracker, to pick the next ticket off the list.


Next JSON: type classes, building blocks
got a Int, List[A], you can bash them together and do list[int]
(Comment this is the same pattern for the database)

TODO

LUNCH

# Back to the issue tracker...

Next: composition in doobie.  Add auditing to a table.
Every change should be recorded.

CAKE and more COFFEE

TODO

# Back to the issue tracker

Next: migrations, http PATCH on a customer

TODO

HOME TIME.

# Summary

- we have seen...
- they are not hard ideas
- they help us with change
- we like scala in particular because of reasons
- Libraries all follow this, not designed to work together.
- if you liked the ideas, and want to play with the code, $URL

-ends-


## GENENAL POOL OF IDEAS

- big things from small things; which is good as we only know how to build small things.
- combining things e.g., shape and music examples.
- drawing: red triangle on top of brown square, vs. turtle 
- types: option, then another option
- adt  / structural recursion / exhaustive matching - description or representation / doing
- json (building blocks: Int, List[A], you're done)
- composition
- patch
- A => B, B => C, A => C
- ...a bit like Task, asynchronous. Example: average server load. Tasks run a function through them.
- perhaps, map, flatMap, etc - add 1 to an Option[Int]

