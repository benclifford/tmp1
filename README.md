# 0. Introduction

We're going to build a very simple room booking server in Haskell, and
expose the API through an HTTP interfaces that roughly corresponds with
some REST principles (* although not all of them).

We won't be building a pretty user interface to go on the front, and
several other features are going to be missing. But hey we only have a
few hours.

This tutorial is a bit of a variety pack, introducing a whole pile of
assorted techniques and libraries, none in much depth.

# 1. Create a new `stack` project, build and run it.

I'm going to assume here that you have the prerequisites installed.

You can use `stack` to create a new skeleton for a Haskell project that
doesn't do much, by running `stack new` and choosing a name for your
project. I've called it `booking` here.

```
$ stack new booking
...
````

Next you can ask stack to build it. The first time you do this, stack will
probably download a pile of dependencies - maybe even a new version of the
Haskell compiler.

```
$ stack build 
...
```

If that worked, you will have a skeleton executable that doesn't do much:
it just prints out a simple string:

```
$ stack exec booking-exe
someFunc
```

I'll briefly go over some of the important files that were created:

There are two Haskell files:

* `app/Main.hs`
* `src/Lib.hs`

The executable `booking-exe` was built from `app/Main.hs`, but it made use
of a library, `src/Lib.hs`. The separation is going to be pretty arbitrary
in this tutorial, but if we wanted to share code between several different
executables, we could place it in `src/Lib.hs`, while putting executable
specific code in `app/Main.hs` and other application files.

There are also two packaging files:

* `package.yaml`
* `stack.yaml`

`stack.yaml` controls how stack behaves when building this project, while
`package.yaml` describes how this code can be packages to interoperate with
other code in other projects.

I'll explain changes to these files as we go along.

# 2. Create a simple 'ping' endpoint with `servant`

Our executable `booking-exe` really does very little at the moment. I'm going
to modify the project so that `booking-exe` is a very simple web server that
responds to a single `/ping` URL.

First, modify package.yaml as shown in the diff. We need to modify two
sections:

* We need to add some dependencies. These are Haskell packages that will
be downloaded when we build, and made available for use in our source code.

The two dependencies we add are: `warp` and `servant-server`. Between them,
they provide the code we need for setting up our web server.

We also need to turn on two language extensions: `DataKinds` and
`TypeOperators`. These extensions change the dialect of Haskell to one
which starts to blur the boundaries between types and values.

We could `stack build` if we wanted to now - although it isn't mandatory.
We might see stack download those extensions; but we would still end up
with a `booking-exe` that is as boring as before.

Now we can actually write some Haskell code. This is split into two parts:

First, we're going to write code to run a web server serving an API.

Secondly, we have to actually declare that API.

To run the webserver, we can import some modules from Servant and Warp:

```
import qualified Servant as S
import Servant ( (:>) )
import qualified Network.Wai.Handler.Warp as W
```

Note my preferred import style here: either explicitly name what is being
imported, or import only as a qualified import. The intention there is that
when I look at a name in the program code, I can tell where it was imported
from.

We can replace the main function in `app/Main.hs` with one that runs our
server:

```
 main :: IO ()
 main = W.run 8080 $ S.serve api server
```

If we tried to run this now, we would get a compile error: we haven't
defined either `api` or `server`.

So let's do that now.

There are two parts: `api` and `server`.

`api` is something like a fancy type signature for the HTTP API that we
will be exposing, and `server` is the code that actually implements that
API.

To begin with, we'll have quite a simple API and server:

```
type PingAPI = "ping" :> S.Get '[S.PlainText] String

api :: S.Proxy PingAPI
api = S.Proxy

handlePing :: S.Handler String
handlePing = return "PONG"

server = handlePing
```

That `PingAPI` type is a bit weird looking, if you're used to writing
normal Haskell types: for example, it's got a String in it, which isn't
even usually a type... and usually when you declare a type, you end up
using values of that type. So this is something wierd. Think of it
informally as itself being a constant value that lives in the type system,
rather than a "data type" that has values.

What that type declaration says: The URL `/ping` on our server will be
accessible using an HTTP GET.  It will return plain text, and that plain
text will come from some Haskell code that returns a String.

Then we can get an `api` value using the `Proxy` data type - which provides
a value with no real runtime content (like `()`) but with interesting
compile time content (the weird looking type PingAPI).

The Haskell code that returns a string is `handlePing`: a `Handler` which
returns a String in the most trivial way we can do at the moment.

If we compile this, the compiler will check that the API (which is declared
to be handled by some code that returns a `String`) matches up with the
handler implementation (which really does return a `String`).

Now we can run booking-exe and point a local web browser at our local
machine http://127.0.0.1:8080/ping and hopefully see the response PONG.

If your network setup is ok, you can point a browser on another machine at
this server (although you'll have to change the IP address) and access this
server over the network.



# 3. Implement a Haskell booking system

Now we're going to put aside the web server part of this project for a while
and build a very simple booking system that, to begin with, we'll only
interact with inside a Haskell REPL. The plan is that once we have it working
there, we'll wire it into the webserver we just set up.

I'm going to approach this by making useful data types, and then making
useful operations on those datatypes.

I'm going to make this a really simple booking system: there will be only
one room, and one day, and bookings can only start and end on hourly 
boundaries.

## 3.0 A `Booking` datatype with a smart constructor

The first thing to represent is a single booking - which we will
represent with a `Booking` Haskell type, in a simple record style.

This goes in `src/Lib.hs`:

```
data Booking = Booking {
    _start :: Int,
    _end :: Int,
    _description :: T.Text
  } deriving Show
```

Now we can load this into a REPL, such as `stack repl`, and create
some bookings.

But we can also create bookings that
don't make sense: for example, the end time can be after the start time,
or the numbers can be negative, or later than 24 (= midnight).

So we might always be a little unsure in our code if we should be checking
for invalid cases like that.

One way to solve this: Instead of allowing anyone to call the `Booking`
constructor, we can restrict access so only our library code can call it.
If something else, like our application code, wants to create a booking,
it has to use some kind of helper function exposed by the library, a so-called
"smart constructor". That smart constructor can guard access to the real
constructor, and throw errors if it doesn't like what it sees.

The `Booking` constructor takes some parameters and returns a Booking,
always. But a smart constructor might not return a booking, if it doesn't
like the parameters. What then can it return instead?

We'll use a common pattern of representing errors using the Either type:

```
mkBooking :: Int -> Int -> T.Text -> Either String Booking
```

This smart constructor will return either a Booking (on the right, if the
parameters are "right", ahaha), or if something isn't Right, it is
Left, and we can represent the error there in some error type -
I'm just going to use String here and stick in human readable messages.

Compare that to throwing exceptions. We're declaring the type of errors
that might come back: `String`. But when an error comes back there isn't
some inbuilt change in program flow - the result is just a value, just
as much as a correcting Booking result is.

We can export the smart constructor, and not export the real constructor,
and then at the REPL, try creating bookings.

Note, though, that a booking that comes back is always wrapped inside
a Right constructor - we don't have a function any more that will give
us a pure Booking.

## 3.1 Collections of bookings

How can we store a collection of bookings that people have made?

The simplest type is a list, `type Bookings = [Booking]`

We can create a couple of bookings, and put them into a list.

Remember we can't just get a booking value, though - it is always wrapped
in an either.

So I can't just write:

```
[mkBooking 1 2 "first", mkBooking 3 4 "second"]
```

because that would give us a list of type [Either String Booking] - we would
have a list where each element was either a booking or an error message about
that attempt at booking.

We can use `do` notation:

do { a <- mkBooking 1 2 "first"; b <- mkBooking 3 4 "second"; return [a,b] }

what comes back *isn'* a pure Bookings list - this monadic notation has
wrapped up the Bookings list inside a similar looking Either type.

If we break one of the bookings, for example by giving it a bad end time,
we can see that no list is returned and we get an appropriate error message
instead.


There's some more validity checking we could do in this way: if two valid
bookings overlap in time, then they can't go in a collection of bookings
together. In that case, it wouldn't be the individual booking objects that
would be badly formed, but the list of bookings.

We can apply similar techniques: we can make a new list type for
bookings, with smart constructors for cons and nil; and we can have
those two smart constructors refuse to construct invalid collections of
bookings.

XXXX

3.1 we don't check that bookings can't overlap: for example this succeeds, but I'd like it to fail, because
booking B overlaps booking A.

*Main Lib> do { a <- mkBooking 9 17 "A" ; b <- mkBooking 15 21 "B" ; b1 <- consBooking a [] ; consBooking b b1 }
Right [Booking {_start = 15, _end = 21, _description = "B"},Booking {_start = 9, _end = 17, _description = "A"}]

So lets enhance our smart-cons-constructor to check the new proposed booking against what we have already

In the example, use non-monadic `if`, instead of `when` style validators. hopefully can see how to convert between those two formats.

## 3.2 A database of `Booking`s, stored in Software Transactional Memory (STM)

3.2 Now drive this with STM, to let us have a single in-memory database (that gets reset every time we restart)

*Main Lib Control.Concurrent.STM> db <- atomically $ newTVar [] :: IO (TVar [Booking])
*Main Lib Control.Concurrent.STM> atomically $ readTVar db
[]

or with some helpers:

*Main Lib Control.Concurrent.STM> db <- mkEmptyBookingDB 
*Main Lib Control.Concurrent.STM> atomically $ readTVar db
[]

*Main Control.Concurrent.STM> let (Right r) = mkBooking 1 10 "first"
*Main Control.Concurrent.STM> addBooking db r
Left "new booking overlaps existing booking"

Now all the booking logic that we'll need is implemented in this library, separate from anything to do with the web side of
things.


# 4. An HTTP endpoint to retrieve a `Booking` as a string

6.1
so back to app/Main.hs...

we'll iniitalise a db and hard-code in some example bookings to start with.

we'll make an endpoint called /bookings.txt which will dish out the haskell list of current bookings - so we'll see the
hard-coded example bookings.

in haskell "Show" form - which looks roughly like haskell constructors.


# 5. Returning a booking as JSON, with Generics

6.2 but we don't want this to be haskell specific. I want to send data in a more common wire format. JSON is one common
example of that (XML might be another)

Fortunately there's a library for that - aeson

with a bit of boiler plate we get:

[{"_start":23,"_description":"second","_end":24},{"_start":9,"_description":"first","_end":12}]

which is JSON, based on the constructor fields - with underscores in the name, like in the constructors.

That's a bit awkward - but if we're going to have it do the translation automatically, then so be it.

and very automatic it is - we don't do any explicit coercian from a Booking to some JSON on the wire.

If we were specifying an API properly, maybe we'd want to implement to JSON <-> Booking conversion more
explicitly.


# 6. Capturing part of a URL as a parameter

6.3 now we can capture a field and use it.
in the simplest impl, we now expose the constructors of Booking so that _start is accessible.
Maybe there's a way to expose _start as an accessor without exposing a constructor: we don't have
a problem with people seeing the insides of a booking - just not constructing arbitrary ones...

6.4 the error handling is pretty poor though - if we pick a bad start time, we don't get a very
nice response. we can pattern match and both log on the console something better, and maybe
send a better response - eg an error code

# 7. Creating bookings with an HTTP POST

7. now we can read the bookings as a whole, and individual bookings - in JSON

It would be nice to be able to make a booking too!

Conventionally that is done by an HTTP POST (rather than an HTTP GET) to the collections URL,
supplying the details of the new booking to make.

we won't be able to generate this HTTP POST request directly from browser UI - though it might come
from some javascript executing in a web page.

First lets check we use a different client:

curl --verbose  http://172.17.0.2:8080/bookings

should show the bookings at command line, as in browser.

curl does a GET by default.

now we'll switch it to using POST:

curl --verbose  http://172.17.0.2:8080/bookings

...
< HTTP/1.1 405 Method Not Allowed
...

so letes implement that...

rather this:
curl -X POST -H "Content-Type: application/json" -d '{"_start":3, "_end":4, "_description":"hi"}' --verbose  http://172.17.0.2:8080/bookings

# 8. Smart constructors vs FromJSON

8. ... now we can make bookings with a POST.

Except badness!

That fromJSON instance means we can now construct arbitrary Booking objects - we've exposed a new constructor that is too liberal,
circumventing our desire to expose a safe API...

For example, you can now make this booking:

curl -X POST -H "Content-Type: application/json" -d '{"_start":1, "_end":-1, "_description":"hi"}' --verbose  http://172.17.0.2:8080/bookings

which ends at -1, so it is invalid in two ways: the end time is before the start, and the end time is less than 0.

So we can't really allow this arbitrary Generic FromJSON  for Booking if we want the existence of a Booking object to mean
that it is well formed.

Instead, can we write our own FromJSON instance, which does a bit more checking?
Yes!

FromJSON can already fail in some cases - for example, if you don't specify all the necessary fields for the constructor.

We can see this at the repl, away from the webserver:


import Data.Aeson

an invalid one:
*Main Lib Data.Aeson> eitherDecode "{}" :: Either String Booking
Left "Error in $: key \"_start\" not present"

and a "valid" one that we'd like to be invalid:

*Main Lib Data.Aeson> eitherDecode "{\"_start\":12, \"_end\":-1, \"_description\":\"test\"}" :: Either String Booking
Right (Booking {_start = 12, _end = -1, _description = "test"})

(we've got that same left/right error pattern here... an error string or the "Right" value)


So let's get rid of that broken FromJSON derived instance ... (after removing FromJSON, we'll get a compile error - because we now have no description of how to convert JSON to a Booking)

... and replace it with something more manually written.

We use applicative notation, but instead of applying the parameters to the base constructor, we apply them to the
smart constructor - and then put in the case for correctly working and for failing.

Unfortunately the nice error message we can supply here doesn't go anywhere useful by default, which is a shame.
But at least we can't make invalid bookings now.




questions people asked:

  * can use multiway if instead of monadic either notation? whats the difference?
      so multiway if doesn't need monad instance - could return anything
      - and the control flow is a little simple - it hits the `if` and
      comes out in one place. (a more elaborate version of the two-way
      `if` i already used)
      `Either` monadic form is more complicated: a particular step doesn't
       necessarily cause a failure or a return value - indeed only the
       last step causes a return. but we can cause a failure from deep
       within an expression easily (by using the `Left` action) rather
       than having to wire up manually the responses (see how in the 
       one way `if` statement I test using `overlaps` but then I have
       to use an `or` to combine all the test results together  - that
       is elegant in the particalar case of overlaps and looks nice,
       but if i have lots of different tests, any of which should cause
       a failure, the `Left` form is better. The `Left` form also allows
       error messages to vary by error, which a test combiner using `or` or
       `and` doesn't - because a False value doesn't convey a reason.

     - or accumulate using state monad so that we can return a whole list of
       reasons this is invalid. this was pretty easy to implement with the
       state monad from `mtl` - but this is also applicative: accumulate a
       list of errors, none of which influence each other; then if the
       accumulated state is [], return the Right booking, otherwise return
       Left [accumulated errors]


  * how does this work in a cloud setting?
     you can run as many instances of this as you want and route requests
     to them. what needs to be shared? the database (and STM doesn't
     provide the ability for that - in a more real environment, you'd use
     something like a single postgres install that all these haskell
     processes talk to)

       
     
other thoughts:

 * probably should make fields strict by default? (as a good practice,
   rather than to fix any bug: 

> If a datatype contains data, it should usually be a strict field. If a datatype models control flow, then it should be a lazy field.
  
https://www.reddit.com/r/haskell/comments/9tm84m/how_can_i_become_comfortable_with_laziness_in/e8xfsze

from tokyo:

* emphasis on "moving towards dependent types"

* type level syntax compare to haskell syntax - "values at compile time" vs "values at runtime"
    - in part 1, for constructing ping API and describing what we've done

also show some type errors

took:
PostMan - windows GUI tool for creating/inspecting HTTP post requests

