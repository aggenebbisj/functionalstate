# Functional state

## From mutable to immutable
Note: or the title on JFall Site "Never change state and still get things done"
State can be difficult. 
Concurrent updates can lead to inconsistency, it can be difficult to scale and have you ever tried testing a component with a random element without having to resort to mocking? 
Functional purity can help us here. 
In this talk, we are going to investigate how we can design a pure functional structure that abstracts over state manipulations. 
We will start with a solution that uses mutable state to solve a problem. 
Then we will refactor step by step and eventually transform it to pure functional code that never changes state. 
Familiarity with lambdas is assumed, but no knowledge of functional programming is required. 
Code examples are in Scala, but no advanced language concepts are used, so knowledge of Java 8 is sufficient. 
As an audience, I would get a comparative overview of solving a problem in a traditional way with imperative code and with pure functional structures. 
I would also learn the benefits of purely functional state and its drawbacks.


## Outline

- The problem with mutable state
  - (Shared) mutable state
  - Referential transparency
    - Testable
    - Composable
    - Parallellizable
- Transitioning from mutable to immutable
  - Domain
  - OO approach with mutable state
    - Example testability problems
    - Example composability problems
    - Example parallellizable problems
  - Immutable approach without State Monad
    - Copy on write 	
    - Pros
      - Testable
      - Parallellizable
      - Composable?
  - State Monad
    - Derive from type signature
    - Wiring done automatically
    - Composable  

---

## The problem with mutable state


### Shared mutable state

- Sharing is no fun
- Non-deterministic 
- Side effects
- Hard(er) to reason about 

Note: 
What are stating here about 'Shared mutable state'? Benefits, consequences ....
Sharing is no fun: probable reference to childhood ;-)?
Non-deterministic has to do with concurrent access and bugs caused by concurrency issues
Side-effects(yeah, that basically is a consequence of having shared mutable state which is updated.)
Harder to reason about, true 


### Try calling this function more than once...

```scala
var y = 0

def f(x: Int): Int = {
  y = y + x + x
  y
}
```

`f(f(2)) == 4 + 4 + 4 != f(4)`

Note: see end of presentation for illustrating it in a different way.


### `f(x) shouldBe f(x)`

```scala
def f(x: Int): Int = x + x
```

`f(f(2)) == f(4)`

<aside class="notes">
This sounds logical, but side-effecting functions throws spanner in the works.
Functions that have assignments that change the state of the system,
break this logical sounding/seeing equation.

By the way shouldBe is from the Scala Test framework, and this roughly translates to assertEquals.
</aside>



### Referential transparency

>An expression is said to be  
referentially transparent  
if it can be **replaced**  
with its corresponding value   
**without changing** the program's behavior

<aside class="notes">
Calling a function with the same arguments will return the same result every time.

This means that you have no assignments that change the global change.
This is why immutability is so important.

But it also has more impact. For example on exceptions (try/catch semantics). 
</aside>


### Benefits of immutability

* Easier to reason about
* Testable
* Composable
* Parallellizable

Note: * Modular why? What are the benefits?
And benefits of what??
Looking at the red book (p. 79), they argue that 
the lack of referential transparency implies that the code is not as testable, composable, modular and easily parallelized as it could be.

--- 

# Transitioning from mutable to immutable


## The domain

![](/img/candydispenser.jpg)

Note:
TODO Make image full size, bottom is clipped now
A vending machine from which you can buy candies. 
- The vending machine can process two types of inputs from customers. 
  1. They can insert money 
  2. They can turn the knob to get the candy after they inserted the money.
- The machine can be refilled with a new stock of candies.
- The returns of the machine can be collected, all coins will be extracted. 

---

## The traditional approach

![](/img/object-oriented-software-construction.jpg)

Note: who knows this classic writing?
First edition written in 1988 by Bertrand Meyer, the second edition at a whopping 1200 pages appeared in 1997
"Meyer pursues the ideal of simple, elegant and user-friendly computer languages and 
 is one of the earliest and most vocal proponents of object-oriented programming (OOP). 
 His book Object-Oriented Software Construction is widely considered to be the best work on presenting the case for OOP."
 https://en.wikipedia.org/wiki/Bertrand_Meyer


## A first attempt

```scala
sealed trait Input
case object Coin extends Input
case object Turn extends Input

class Machine(private var _candies: Int,
              private var _coins: Int) {

  // 'Commands' modify machine
  def process(inputs: List[Input]): Int 

  // 'Queries' return information about machine
  def candies = _candies
  def coins = _coins
}
```

<aside class="notes">
TODO kan command nog? Het returnt geen Unit meer nu.
Meyer p. 748 "The features that characterize a class are divided into commands and queries. 
A command serves to modify objects, a query to return information about objects."
p. 749 A function (defined as routine that returns a result) produces a concrete side effect if its body contains any of the following
* an assignment, assignment attempt or creation instruction whose target is an attribute * a procedure call
On p. 751 he formulates the 'Command-Query separation principle' as "Functions should not produce abstract side effects"
</aside>


## Taking it for a spin

```scala
val machine = new Machine(_candies = 5, _coins = 10)

machine.process(List(Coin, Turn))

machine.coins shouldBe 11
machine.candies shouldBe 4
```


## Testing it

TODO maybe something with if statement that changes behaviour after x turns?

---

## Introducing immutability


### Remember?

```scala
sealed trait Input
case object Coin extends Input
case object Turn extends Input

class Machine(private var _candies: Int,
              private var _coins: Int) {

  // 'Commands' modify machine
  def process(inputs: List[Input]): Int 

  // 'Queries' return information about machine
  def candies = _candies
  def coins = _coins
}
```


### Copy on write

```scala
sealed trait Input
case object Coin extends Input
case object Turn extends Input

case class Machine(candies: Int, coins: Int)
  
object Machine {
  def process(inputs: List[Input], machine: Machine): (Machine, Int)
}
```

<aside class="notes">
We have to introduce companion object here. 
Best explained for Java people as an object to hold your static methods. 
We need the methods out of Machine case class to prepare for type signature of State Monad.
</aside>


## Do you spot the bug?

```scala
val m = Machine(candies = 100, coins = 0)
val p1 = List[Input](Coin, Turn, Coin, Coin, Turn)
val (m2, c) = Machine.process(p1, m)
val p2 = List[Input](Coin, Coin, Turn)
val (m3, c2) = Machine.process(p2, m)
```


## Cons

- Manual wiring of state

---

## Functional State


## Let's look at the type signature

`def process(inputs: List[Input], machine: Machine): (Machine, Int)`

===

`def process(inputs: List[Input])(machine: Machine): (Machine, Int)`


## `Machine => (Machine, Int)`


# `S => (S, A)`

Note:
`S1 => (S2, A)`


## Let's wrap this in a class

```scala
class State[S, +A](f: S => (S, A)) {
  def run(initial: S): (S, A) = f(initial)
}
```

<aside class="notes">
Without combinators the class does not do much of course. We need the class because unlike JavaScript we cannot add methods to functions.
</aside>


## Doing the mechanical refactoring

```scala
def process(inputs: List[Input])(machine: Machine): (Machine, Int) = {
  ...
}
```

becomes

```scala
def process(inputs: List[Input]): new State[Machine, Int](machine =>
  ...
)
```


## The implementation

```scala
object Machine {
  def process(is: List[Input], m: Machine): (Machine, Int) =
    is.foldLeft((m, m.candies)) { case ((m, c), input) => 
      input match {
        case Coin => (m.copy(coins = m.coins + 1), m.candies)
        case Turn => (m.copy(candies = m.candies - 1), m.candies)
      } 
    }
}
```

<aside class="notes">
Possibly skip this slide and the next. Could be too complex with the foldLeft.
</aside>


## The implementation with State

```
object Machine {
  def process(is: List[Input]): State[Machine, Int] =
    new State( m =>
      is.foldLeft((m, m.candies)) { case ((m, c), input) =>
        input match {
          case Coin => (m.copy(coins = m.coins + 1), m.candies)
          case Turn => (m.copy(candies = m.candies - 1), m.candies)
        }
      }
    )
}
```

---

## Combinators

>"A combinator is a higher-order function 
>that uses only function application and 
>earlier defined combinators 
>to define a result from its arguments."

Note: from https://en.wikipedia.org/wiki/Combinatory_logic


## `map`

```scala
class State[S, +A](f: S => (S, A)) {
  def run(initial: S): (S, A) = f(initial)
  
  def map[B](transform: A => B): State[S, B] =
    new State[S, B](s => {
      val (s, a) = run(s)
      (s, transform(a))
    })
  }
```


## Example

```scala
val inputs = List[Input](Coin, Turn, Coin, Coin, Turn)

val program = Machine.process(inputs)
                     .map(candies => s"Candies left: $candies")

program.run(Machine(candies = 100, coins = 0))
```

```
$ Candies left: 98
```


## `FlatMap`

```scala
class State[S, +A](f: S => (S, A)) {
  def run(initial: S): (S, A) = f(initial)
  ...
  def flatMap[B](g: A => State[S, B]): State[S, B] =
    new State(s0 => {
      val (s1, a) = run(s0)
      g(a).run(s1)
    })
```


### `FlatMap` chains two state functions

```                                                    
                     -----            -----         
                     |   | ->  m1  -> |   | -> m2 
               m0 -> | f |            | g |         
                     |   | ->   a --> |   | -> b
                     -----            -----         
```


## Using for comprehensions

```scala
val initial = Machine(candies = 100, coins = 0)
val p1 = List[Input](Coin, Turn, Coin, Coin, Turn)
val p2 = List[Input](Coin, Coin, Turn)
```

```scala
val program = Machine.process(p1).flatMap(_ => Machine.process(p2))
val (m, c) = program.run(initial)
```

```scala
val program = for {
  _ <- Machine.process(p1)
  c <- Machine.process(p2)
} yield c

val (m, c) = program.run(initial)
```

---

## Buying a candy

```scala
def input(input: Input): State[Machine, (Int, Int)] =
  State { m =>
    val m1 = m.process(input)
    ((m1.coins, m1.candies), m1)
  }
```


## Refilling the machine

```scala
def refill(newCandies: Int): State[Machine, (Int, Int)] =
  State { m =>
    val m1 = m.copy(locked = true, newCandies + m.candies)
    ((m1.coins, m1.candies), m1)
  }
```


### Collecting
```scala
  def collect(): State[Machine, (Int, Int)] =
    State { m =>
      val m1 = m.copy(locked = true, coins = 0)
      ((m1.coins, m1.candies), m1)
    }
```


## State

<aside class="notes">
Erik Meijer: "On the other hand, radically eradicating all effects—explicit and implicit—renders programming languages useless." (E. Meijer, ACMQueue) 
</aside>


![](/img/Utrecht_Moreelse_Heraclite.JPG)
<aside class="notes">
Image: "Heraclitus by Johannes Moreelse. The image depicts him as "the weeping philosopher" wringing his hands over the world, and as "the obscure" dressed in dark clothing."

Heraclitus of Ephesus, 535 – c. 475 BC,  most known quotation is 'Panta rhei' or everything flows.
"Ever-newer waters flow on those who step into the same rivers." or "All entities move and nothing remains still". 
Basically stating that the world is constantly changing.
"All entities move and nothing remains still"

image src: https://en.wikipedia.org/wiki/Heraclitus#/media/File:Utrecht_Moreelse_Heraclite.JPG

Random Number Generation is a typical case of programming with side effect. 

</aside>


### Random number generation
```scala
def rollDice: Int = {
    val rng = new scala.util.Random
    rng.nextInt(6)
}

"expect valid dice" in {
  val beValidDice = be > 0 and be <= 6
  rollDice should beValidDice
}

```

Note:
A method on Random with no parameters delivers a pseudo random number. It must depend on some form of internal/global state!
Given the seed a Random will deliver the same sequence of numbers, but with each invocation the value changes. 
This method is NOT referential transparent!

What if we roll the dice? and we want to assert the outcome? Is this correct? Or should we do this 1.000 times.
But wait, there is a bug. This method has an off-by-one error.
From the API of nextInt: Returns a pseudorandom, uniformly distributed int value between 0 (inclusive) and the specified value (exclusive), 
drawn from this random number generator's sequence.

How can we fix this? Keep track of the number of invocations of Random? No! We will avoid using side-effects.


```scala
trait RNG {
	def nextInt: (RNG, Int)
}
```

Note: 
"The key to recovering referential transparency is to make the state updates explicit."
This nextInt method return a random generated number and the new state, leaving the old state unmodified.
"In effect, we separate the concern of computing what the next state is from the concern of communicating the new state to the rest of the program."
This can be expanded to basically all statefull API's. 
Just let the API compute the next state and return the value next to the new state, leaving the old state intact.


```scala
val original: RNG = ...

val (value, nextRng) = original.nextInt

(value, nextRng) shouldBe original.nextInt

```

Note: say we create a Random Number Generator instance.
nextInt will generate a tuple of a random generated value and the new Random Number Generator.
Even if we take the original RNG and call nextInt again, we get back the same response as on the first call.
It has become referential transparent again!


```scala
S => (S, A)
```

Note: Should we move these slides to the composability part?


```scala
S => (S', A)
```


---

## Constructing and transforming


```scala
def unit[A](a: A): Rand[A] = rng => (a, rng)

// Always return 1, no matter the state, rng never changes
unit(1) => rng => (1, rng)
```


```scala
def map[A,B](stateGenerator: RNG => (RNG, A))(f: A => B): RNG => (RNG, B) = rng => {
  val (a, rng2) = stateGenerator(rng)
    (f(a), rng2) // transform value here
  }
  
val newF = unit(1).map(x => x + 1) // function that always returns 2

newF(newRng) // will return 2
```



---

## Composability: Combining state actions


### map2: combining two state actions

```                        
                        -----
                        |   | ------------------> rngB
       -----            | B |            -----
       |   | -> rngA -> |   | -> b ----> |   |
rng -> | A |            -----            | f | -> c
       |   | -> a ---------------------> |   |
       -----                             -----
```


### sequence: combining multiple state actions

```scala
def sequence[A](fs: List[Rand[A]]): Rand[List[A]] = {
  fs.foldLeft(unit(List.empty[A])) { (acc, rand) => 
    map2(rand, acc)((rand, acc) => rand :: acc)
  }
}
```


## Nesting state actions


```scala
def flatMap[A,B](f: Rand[A])(g: A => Rand[B]): Rand[B] = 
  rng => {
    val (a, rngA) = f(rng)
    g(a)(rngA)
  }
```


```                                                    
       -----                          -----         
       |   | -> rngA                  |   | -> rngB 
rng -> | f |             a -> rngA -> | g |         
       |   | -> a                     |   | -> b
       -----                          -----         
```


```                                                    
       -----            -----         
       |   | -> rngA -> |   | -> rngB 
rng -> | f |            | g |         
       |   | -> a ----> |   | -> b
       -----            -----         
```

---

## Candy Dispenser Example


### The domain

```scala
sealed trait Input
case object Coin extends Input
case object Turn extends Input

class Machine(locked: Boolean, candies: Int, coins: Int) {
  def process(input: Input): Machine = ???
}
```


### Using a state function (?? What should be the better name)
```scala
  def input(input: Input, m: Machine): (Machine, (Int, Int)) = {
      val m1 = m.process(input)
      (m1, (m1.coins, m1.candies))
    }

      val machine = Machine(locked = true, candies = 5, coins = 10)
      val (m1, _)                = input(Coin, machine)
      val (m2, (_, candies2))    = input(Turn, m1)
      val (m3, _)                = refill(candies2, m1)
      val (m4, (coins, candies)) = collect(m3)

```
<aside class="notes">
On the line with m3 there is an error, m1 is used as argument, but m2 should be used.
The types are correct, so this sucks.
You need to thread the state through all the different state changing functions
</aside>


### Introduction of State
```scala
case class State[S, +A](run: S => (A, S))

```


### State

```scala
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] = ???
     
        

  def flatMap[B](f: A => State[S, B]): State[S, B] = ???
        
            
    
```

Note: For combining State operations we need higher-order functions map and flatMap.
Map takes a function which takes an A and returns a B, which in turn returns a new State and B.
FlatMap



### Map
```scala
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] = State { s =>
    val (a, s1) = run(s)
    (f(a), s1)
  }

  def flatMap[B](f: A => State[S, B]): State[S, B] = ???
```


### Map Usage
```scala
   
        input(Coin)
          .map { case (coins, _) => s"Returns are EUR $coins" }
```


### Map Usage
```scala
      val pipeline: State[Machine, String] =
        input(Coin)
          .map { case (coins, _) => s"Returns are EUR $coins" }
          
      val (message, _) = pipeline run machine
      
      message shouldBe "Returns are EUR 11"

```



### FlatMap
```scala
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] = 
    State { s =>
      val (a, s1) = run(s)
      (f(a), s1)
    }

  def flatMap[B](f: A => State[S, B]): State[S, B] = ???
```


### FlatMap
```scala
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] = 
    State { s =>
      val (a, s1) = run(s)
      (f(a), s1)
    }

  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State { s =>
      val (a, s1) = run(s)
      f(a).run(s1)
    }
```


### Usage FlatMap
```scala
   
  input(Coin)
    .flatMap(_ => input(Turn))
```


### Usage FlatMap
```scala
val pipeline: State[Machine, (Int, Int)] =
  input(Coin)
    .flatMap(_ => input(Turn))

val ((coins, candies), _) = pipeline run machine
coins shouldBe 11
candies shouldBe 4
```



### Combine in Maintenance
```scala
def maintain(candies: Int): State[Machine, (Int, Int)] = {
  refill(candies)
    .flatMap(_ => collect())
}
```
<aside class="notes">
Combining the two is just flat mapping the functions, diagram to illustrate ??
</aside>


### For-comprehension
```scala
def maintain(candies: Int): State[Machine, (Int, Int)] =
  for {
    _ <- refill(candies)
    s <- collect()
  } yield s
```
<aside class="notes">
Or using a For-comprehension which compiles to the same code.
</aside>


---

## Examples transforming functions

```bash
scala> val f: Int => Int = i => i
f: Int => Int = <function1>

scala> f(3)
res0: Int = 3

scala> f(4)
res1: Int = 4

scala> def map(f: Int => Int)(tranform: Int => String): Int => String = {
     |   i => transform(f(i))
     | }
map: (f: Int => Int)(tranform: Int => String)Int => String

scala> map(f)(i => "Hello " + i)
res2: Int => String = <function1>

scala> val f2 = map(f)(i => "Hello " + i)
f2: Int => String = <function1>

scala> f2(3)
res3: String = Hello 3
```

---
### Random number generation
```scala
def rollDie: Int = {
    val rng = new scala.util.Random
    rng.nextInt(6)
}
```


```scala
trait RNG {
	def nextInt: (RNG, Int)
}
```


```scala
S => (S, A)
```


```scala
S1 => (S2, A)
```


```scala
type Rand[+A] = RNG => (RNG, A)
```


```scala
trait RNG {
	def nextInt: (RNG, Int)
}

RNG => (RNG, Int)

def int(rng: RNG): (RNG, Int) = rng.nextInt

val int: RNG => (RNG, Int) = rng => rng.nextInt

$ int(someRNG)

val int: Rand[Int] = rng => rng.nextInt
```


<aside class="notes">
Hier moet testability nog aan de orde komen.
</aside>
