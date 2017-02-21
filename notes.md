# Hello

Hello everyone.
Thanks for inviting us here.
I'm Richard, this is Miles, we work for Underscore.
Underscore does software development,
using a programming language called Scala,
and as well as development we do consultancy around code and architecture,
and training.
We work with all sorts of companies all over the world,
and Miles and I happen to be based down the road in Brighton.

# Intro

We're here to talk about modern software development.
And the way we're going to do that
is to give you a day in the life of a functional programmer.

We'll go through the kinds of things you do writing software,
but through the eyes of a functional programmer.

And we're doing this because for us modern means functional programing with types.

# FP + T

And we'll get on to what that means in a moment.

For us it's particular, liberating, and exciting style of development
We're not talking about being modern because of any particular product or architecture.
We're not modern because we're using React,
or MongoDB, 
or whatever is hot right now.

Rather, it's the ideas and techniques from functional programming that matter.
They give software developers a huge helping hand in every-day programming.

# Why now

FP is about programming using functions,
and using functions as values to other function,
recursion,
constructing nested data structures,
lots of ideas stolen from maths.

We especially like functions that produce no side effects.
What's been realised is that that style is a great fit to scaling in parallel
and across large data sets.
If you've heard of map reduce, you'll know what we mean.

The core ideas have been around for decades.
So why are we excited about this now?

# Modern compilers

It's because modern compilers have powerful type systems.
We're familiar with compilers and languages that catch errors for us.
You gave me a string but I wanted an integer kind of errors.
That's great, but we can do a lot more than that.
We can get the compiler to do some of our work for us,
which lets us write smaller chunks of code.

# Take away points

There are two main things I hope you'll see from this.

The first is that functional programming is for everyone.
Some of the things we're going to show you are simple.
But they have weird names. 

The second thing we want you to watch out for is that these techniques make change easier.
And it does that by making your code better modularized and easier to reason about.

# Morning

Let's start with the daily stand up.
The team has been building a web service.
It's a JSON based API behind a customer management system.
This is all asynchronous, but basically a pretty old style CRUD application.

# API

The API lets you list customers, add customers, update customers.
Simple stuff.

All we have at the moment is customers. And a customer has an id, a name, and a phone number.

# New Ticket

We've picked up a new ticket, and that's to add a subscription indicator to a customer.

Sounds like we're adding a field to a record.
Not too tricky.

# More details

We dig a little deeper and find out that a customer might have:

- No subscription
- An Active Subscription
- Lapsed Subscription

We're going to have to show that in JSON,
store it in the database,
write a state machine to manage transitions between the states.

How could we do that? Instantly a functional programmer says:

> This is a job for an....Algebraic Datatype

I did warn you there were some weird phrases.

This is an incredibly simple idea, you'll wonder what the fuss was about.
We're going to separate the representation from the behaviour.

# Code

We'll see some Scala syntax now.
We're not here to teach you the language.
You're all smart people, you'll get the general idea.
Feel free to stop and ask questions, but don't worry about the details too much.
We'll give you the code so you can explore it in more detail if you want to. 
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
This is pretty simple

# Change

It gets a little more interesting when the inevitable happens.
Something will change.

In this case, we know how to fix this.
We have another case for a subscription.
And it's a simple one.

So we go back to our code.
And what we need to do is add in a case.

We can do that easily enough, 
but if we don't change our code,
it won't compile.
We'll be told each place where we need to consider the new kind of subscription.

And that's really nice, because we can mess with our data type, 
and have the compiler point out each and ever situation we need to re-examine.

BTW, this might look like a feature of Scala.
That we can match and represented sealed cases.
But it's a feature common to many functional programming languages.
Which is to say, it's a technique that has been around a while,
and works for people.

# Versatility

Exhaustivity check is one reason we like ADTs and SR.
Another reason is the versatility.

For example, the code we have so for is okay.
Really, we probably want to do things.

- Like maybe send a nagging email if the subscriber has lapsed.
- Or send an incentive if the customer is expiring soon.

So what does the functional programmer do about that?
What we're not going to do is mix up side effects that change the world,
like sending an email, from our business logic.
Instead we isolate the side effects, and move them as far out as we can to the edges of our code.

We can do that with another ADT

We can change our transition to a customer at a particular date,
and produce some kind of action.

Maybe there's no action. No change to be made.
Or maybe we want to send them some kind of encouragement to renew.
Whatever we want to do.

Transition itself would still be a pure-function.
I'm not going to fill the details of the code in.
It's the same kind of thing you've already seen: match of the cases, produces some value.
It takes a customer and produces an action.
And you can still unit test that as much as you like.

When it comes to sending the email, 
we can separate that out.
A method perform takes an action and returns Unit.
Unit is Scala's representation for no value.
It's signalling there's a side effect going on.

The body of perform would have to match on all the kinds of action.
Again, I won't bore you with the details.
It's the same as you've already seen.

Which is kind of the nice thing about ADTs.
You do a bit of up-front thinking about the various cases.
Action, Subscription.

Then writing the code to handle them is something you can just hammer out on the keyboard,
knowing the compiler has your back if you miss a case.

# Coffee and reflection

So far in our day in the life, we've done some work on the ticket.
We've written maybe 10 lines of code, so it's definitely time for some coffee and cake.

That's one technique a functional programmer uses.
And they'll use it every day.

We like ADTs for their safety.

We like them for their versatility.
They occur everywhere. If you've heard of Option or Optional types,
that's an ADT with two cases: Some and None.
Lists are ADTs, again with two cases: 
the empty list, and a "cons", which is a value and another list.
And you can pattern match on them.

Separating a representation from doing it
Helps isolate side effects
...makes change easier

# Back to work

The next step is that we want the subscription to appear in our JSON.  Something like this:

The punchline here is that we don't have to do anything.
We're using a library which can automatically figure this out for us.
But the principle it uses for this is a general one.
So we'll recreate the basics here,
because it's another pattern that appears again and again.

In particular, we create tiny building blocks,
and combine them up into bigger things.

This is important, because writing small things tends to be relatively easy, which makes development easier.

# JSON

We have a value we want to turn into a valid JSON string we can send back to a web browser.


In the case of an integer, it's pretty easy:

```
7.toString
```

In the case of a string, it's similar but we need quote marks:

```
"'" + "Hello" + "'"
```

In both these cases we take a value and return a string representation.  That's a function:

```
 Int => String
 String => String
```

and in general:

```
 T => String
```

Let's encode that and give it a name:

```
trait JsonFormat[T] {
  def format(value: T): String
}
```

Here we are saying, there's this abstract thing called a JsonFormat.
When you make a json format, you make it for a particular type, and the contract is you're given that type and you get back a string.

Let's create some examples of it:

```
val intFormat = new JsonFormat[Int] {
  def format(value: Int): String = value.toString
}

val stringFormat = new JsonForamt[String] {
  def format(value: String): String =
     "'" + value + "'"
}
```

and we can write a general purpose formatter:

```
def outputJson[T](value: T)(formatter: JsonFormat[T]) =
   formatter.format(value)

outputJson(42)(intFormat)
```

Now this looks a bit ... pointless. To output json, you give me a value, and a formatter for that value, and we format it.
We do this, because in Scala we can ask the compiler to prove it can format something. We'd say:

[change to implicit]

Now we have something a bit more interesting.
In particular, this means the compiler will check that when we ask it to output some Json, someone has created a formater for that thing.

What makes this powerful, is that we can use this to build bigger structures.

How about we want to convert a list of integers to Json.
Or a list of strings.
We don't want to have to write a List[Int] formatter.
We want to write a list formatter and have it work with anything.

Well, not quite anything.
Anything that can be turned into JSON.

So we can write:

```
implicit def listFormat[T](implicit formatter: JsonFormat[T]) = new JsonFormat[List[T]] {
  def format(values: List[T]): String = {
    val formattedValues = for {
      v <- values
      } yield formatter.format(v)
    "[" + formattedValues.mkString(",") + "]"
  }
}

outputJson(List(1,2,3))
```

and

```
outputJson(List( List(1,2), List("a","b") )
```

So this is incredibly powerful.
We have written small functions,
and combined them to create much larger capabilities.


We refer to this as a type class, and there's a simple routine for creating them.

- give the type class a name (JsonFormat) and define a method
- implement the handful of simple instances (e.g., for Int, String)
- use the primitives to build bigger structures

This is used all over the place.

- You want to parse or format Json
- Read or write data in a database
- You want to read Excel and CSV
- Convert from one format to another

We won't go into it now, but you can imagine how this works for something like a Customer:

```
case class Customer(
 id   : Long
 name : String,
 sub  : Subscription
)
```

We can take the fields in turn and ask if there is a JsonFormatter for them. Probably is for an Long, and a String, and for Subscription we can recurse and do the same thing, and if we have converter for a subscription, we can finally convert the whole customer.

# LUNCH

We're good chunk of time into our day.
Over lunch we'll say we've encoded a problem with an ADT and we've used a type class for convert it to Json.
And you know what those things mean.

[mention typelevel libraries work together not designed to but use these patterns]

As an example, show doobie:

```
sql""" select name, phone, id from customer """.query[Customer]
```

string -> varchar
long -> 
....fields in customer

# Back to the issue tracker...

[[ IF IT LOOKS LIKE THERE IS TIME...
Next: composition in doobie.  Add auditing to a table.
Every change should be recorded.
]]


# PATCH

Let's pick up another ticket.

> As a customer, I want an easy to use interface to my details, so that I can update some or all of my details.

So this is all about updating a customer record.
Once upon a time, this would be a form, you'd get all the details, and you'd write the changes to the database.
We don't tend to do that anymore.
Instead, we like rich, interactive pages, where when I make a change, that change is saved.

For our API this means we might get the name of the customer to update.
Or we might get the phone number.
Or we might get both.

Whatever we get, we need to update it. How are we going to do that?

Let's take a look at the API code base:

```
class CustomerAPI(db: Database) {

  val service = HttpService {

    case GET -> Root / "customers" =>
      Ok(db.list.map(_.asJson))

    case GET -> Root / "customers" / IntVar(id) =>
      db.find(id).flatMap {
        case Some(customer) => Ok(customer)
        case None           => NotFound()
      }

  }
}
```

This is a library we're using called http4s.
We don't need to dwell on much of this, but I'll point out a couple of things.

We're exposing a web service.  You case see we're using the matching (structural recursion) we've seen before.
In this case, rather than matching on a Subscription or Action, we're matching on an encoding of a request.
There's been some nice use of the `/` symbol here, to make these routes easy to read.
But this really is the same pattern matching you've seen already.

It could have been something like:

```
case (GET, Request("customers")) => ...
```

...but the http4s team have made it easier to read.
So if we ask for /customers we get a list of all the customers as a 200 response.
If we ask for a particular customer ID, we try to look it up in the database.
That will optionally find a record, and we match on that to produce the appropriate response.

How are we going to take an update on any field.
We're certainly not going to write an endpoint for each possible change:

```
case POST -> Root / "customers" / IntVar(id) / "name" => ...
case POST -> Root / "customers" / IntVar(id) / "phone" => ...
...
```

Not only is that teadious and a maintenance pain, it also wouldn't allow us to update both fields at the same time.
A better fit here would be a JSON Patch.

We're going to receive some updates to some fields. 
A message like this:

```json
{
  "name" : "Robert"
}
```

or


```json
{
  "name" : "Robert",
  "phone : "+1 555 555"
}
```

[not rfc6902 compliant - see diffusion lib, perhaps]

The question is: how to handle that
We're going to pause for a second a think about what we want to do as a function.

We want to take a customer record, take a patch, and merge them together to get an new customer record.

```
(Customer, Patch) => Customer
```

How can we represent the patch?
We want this to be safe, and easy to change.

We could code this up by hand and do something like this in pseudo code:

```
if (json contains "name") customer.copy(name = json("name"))
if (json contains "phone") customer.copy(phone = json("phone"))
```

But that is horrible, error prone, and doesn't do what we want anyway.
If you change the name, we need to remember that, and then also change your phone number.

So, no, we're not doing that.
What we'll do is create a proper representation for the change:

```
case class Customer(
  id    : Long,
  name  : String,
  phone : String
)

case class CustomerPatch(
  name  : Option[String],
  phone : Option[String]
)

def merge(c: Customer, p: Patch): Customer = ???
```

We're not going to hand-write that merge function.
I'm going to outline the approach to how we write code to do this.

What we want to be able to do is:

```
trait Merge[A, P] {
 def merge(old: A, patch: P): A
}
```

You'll probably recognize this now as a type class.
And if so, you know how we're going to solve the problem.

We start with a basic definition:
If our customer contained just a string, under what situation can we update it?
We can do that if the patch contains an optional string.

```
implicit def optionMerge[A]: Merge[A, Option[A]] = new Merge[A, Option[A]] {
  def merge(current: A, patch: Option[A]) = patch match {
    case Some(updated) => updated
    case None          => current
  }
}
```

How do we get to merge a customer and a customer patch from that.
In outline, it goes like this:

- What the types of the field in the patch and the customer?
- Take the first field of the patch and customer
- As the compile to (implicitly) find a way to merge them.
- Merge the rest (which recursively takes us back to the start)
- Combine the updated values back together

The code to do this is getting advanced.
But the principles at play are the ones we've seen.

By analogy, let's suppose our customer and patch were a list of values:

```
def merge[C,P](customer: List[C], patch: List[P])(implicit patcher: Merge[C,P]): List[C] =
  if (customer.isEmpty) Nil
  else patcher.merge(customer.head, patch.head) :: merge(customer.tail, patch.tail)

// Not complete, much nicer ways... zip, fold, etc
// Cannot do hetero types etc

merge( List("Bob", "555-123"), List(None, Some("555-555")) )
merge( List("Bob", "555-123"), List(Some("Robert"), None) )
```

Combining small things together to make a bigger thing.

The example code includes this as a library called bulletin, 
from our colleague Dave Gurnell.
So you can dig around the source and see what it does.
In fact, it's a bit more fancy than I've shown,
as it also matches up the names of the fields.

All of this, by the way is at compile time.
No run-time reflection.
If the program compiles you will merge the values.

So we can add to our service now:

[build this up]

```
case req @ PATCH -> Root / "customers" / IntVar(id) =>
  req.decode[CustomerPatch] { patch =>
    db.find(id).flatMap {
      case None           => NotFound()
      case Some(customer) => 
        val updated = merge(customer, patch)
        Ok(db.update(updated))
    }
  }
``

with

```
val merge = bulletin.AutoMerge[Customer, CustomerPatch]

case class CustomerPatch(
  name  : Option[String],
  phone : Option[String]
)
```

If the types change, compile error
In this implementation, if the names don't match, compile error.


HOME TIME.

# Much more not touched on

- basic combinators
- functors, monads, etc

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

