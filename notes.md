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

(why now... Powerful and modern type systesm...compiler)

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
Really, we probably want to do things.

- Like maybe send a nagging email if the subscriber has lapsed.
- Or send an incentive if the customer is expiring soon.
- ...

So what do we do? 
It looks like we're going to be mixing in side effects like sending an email into our code.
We don't want to do that, we want to isolate those areas from the business logic so we can easily understand it and change it.
What we do is follow the same pattern again.

Yes, it's another ADT!

```
sealed trait Action
case object NoAction extends Action
case class EncourageRenewal(c: Customer) extends Action
case class Expire(c: Customer) extends Action
case class Nag(c: Customer) extends Action

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

You can see the pattern here.
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
  case Nil        => "No numbers"
  case x :: rest => s"The first number is $x"
}
``

Again, two cases: the empty list, and then a value and another list
which happens to be expressed as `::` in Scala.

# Coffee

So far in our day in the life, we've done some work on the ticket.
We've written maybe 10 lines of code, so it's definately time for some coffee.

We can reflect on what we've done so far:

- We saw ADTs
- Separating a representation from doing it
- Helps isolate side effects
- Common pattern used over and over.
- Compile tells you if you've missed a case.

...makes change easier

...super useful in day-to-day development.


# Back to work

The next step is that we want the subscription to appear in our JSON.  Something like this:

[json example of a customer with a sub]

The punchline here is that we don't have to do anything.
We're using a library which can automatically figure this out for us.
But the principle it uses for this is a general one.
So we'll recreate the basics here,
because it's another pattern that appears again and again.

In particular, we create tiny building blocks,
and combine them up into bigger things.
This is important, because writing small things tends to be relatively easy, which makes development easier.

We have a value we want to turn into a valid JSON string we can send back to a web browser.

[maybe show json spec sheet from http://json.org/ ]

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
We do this, because in Scala we can ask the compiler to proove it can format something. We'd say:

```
implicit val intFormat = new JsonFormat[Int] {
  def format(value: Int): String = value.toString
}

def outputJson[T](value: T)(implicit formatter: JsonFormat[T]) =
   formatter.format(value)

outputJson(42)
```

Now we have something powerful and safe.
In particular, this means the compiler will check that when we ask it to output some Json, someone has created a formater for that thing.

[maybe show it failing on string until we add string implicit]

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

