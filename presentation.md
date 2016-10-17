# Functional state

## From mutable to immutable


## Outline

- Why?
  - What is the problem?
    - f(x) shouldBe f(x)
    - side effects (assignments that change the state of the system)

  - How does Functional programming solve this?
     - side-effect free (substition model)
        - No temporal couplings
        - Fewer concurrency issues
        - No asking what;s the state?

- But how can anything happen if you never do anything?
  - State changes still happen

- Example
  - Random numbers (SecureRandom?)
  - Candy dispenser?
  - [Traffic light?](http://timperrett.com/2013/11/25/understanding-state-monad/)

- Advantages
  - You always know the State that was used to create the next value
  - You can share an instance 
  - You can flatMap etc. making it more composable
- Performance / Functional/Persistent Data Structures
- 
---

## Traditional approach


![](/img/object-oriented-software-construction.jpg)

Note: who knows this classic writing?
First edition written in 1988 by Bertrand Meyer, the second edition at a whopping 1200 pages appeared in 1997
"Meyer pursues the ideal of simple, elegant and user-friendly computer languages and 
 is one of the earliest and most vocal proponents of object-oriented programming (OOP). 
 His book Object-Oriented Software Construction is widely considered to be the best work on presenting the case for OOP."
 https://en.wikipedia.org/wiki/Bertrand_Meyer


### Object-oriented software construction

>When objects take over, their former masters, the functions, become their vassals.

> p. 684
Note: Bertrand argues that the top-down functional decomposition has deficiencies and introduces as a design rule 
the 'Law of inversion' stating that "If your routines exchange too many data, put your routines in your data".
Note that he is not talking about functions in the sense of functional programming, 
but on a functional, top-down solution


### The object-oriented domain

```scala
sealed trait Input
case object Coin extends Input
case object Turn extends Input

class Machine(private var _candies: Int,
              private var _coins: Int) {

  // commands modify objects
  def process(input: Input): Unit
  def collect(): Unit
  def refill(newCandies: Int): Unit 

  // 'Queries' return information about objects
  def candies = _candies
  def coins = _coins

```
<aside class="notes">
Domain: a vending machine from which you can buy candies. 
The vending machine can process two types of inputs from customers. 
They can insert money and they can turn the knob to get the candy after they inserted the money.
The machine can be refilled with a new stock of candies.
And the returns of the machine can be collected, all coins will be extracted. 

Note: Meyer p. 748 "The features that characterize a class are divided into commands and queries. 
A command serves to modify objects, a query to return information about objects."
p. 749 A function (defined as routine that returns a result) produces a concrete side effect if its body contains any of the following
* an assignment, assignment attempt or creation instruction whose target is an attribute * a procedure call
On p. 751 he formulates the 'Command-Query separation principle' as "Functions should not produce abstract side effects"

</aside>


### Object-oriented approach

```scala
val machine = Machine(candies = 5, coins = 10)

machine.process(Coin)
machine.process(Turn)

machine.coins shouldBe 11
machine.candies shouldBe 4
```

Note: Do we want to say anything about the apply-method in the companion object?
---

## What is the problem?



### What is the problem?

```scala
f(x) shouldBe f(x)
```

<aside class="notes">
This sounds logical, but side-effecting functions throws spanner in the works.
Functions that have assignments that change the state of the system,
break this logical sounding/seeing equation.

By the way shouldBe is from the Scala Test framework, and this roughly translates to assertEquals.
</aside>


### What is the problem?

```scala
f(x) shouldBe f(x)

def f(x: Int): Int = x + x
```

<aside class="notes">
With pure functions, functions without side-effects this would be true.
We have a simple function that doubles it argument.

An expression is said to be referentially transparent if it can be replaced with its value
without changing the behavior of a program
(in other words, yielding a program that has the same effects and output on the same input).
</aside>


### Side effects

```scala
f(x) shouldBe f(x)

var y = 0

def f(x: Int): Int = {
  y = y + x
  y
}
```

<aside class="notes">
In this case there is an assignment to some outer scope, which changes the outcome for all future invocations.
This function is not pure, it has side-effects and makes the program a lot harder to reason about.
</aside>


## Referential transparency


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

---

### Benefits

* Testable
* Composable
* Parallellizable

Note: * Modular why? What are the benefits?
And benefits of what??

--- 
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
<aside class="notes">
Domain: a vending machine with candies. The vending machine has two inputs for customers. 
They can insert money and they can turn the knob to get the candy after they inserted the money.
</aside>


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
<aside class="notes">

</aside>


### Buying a candy
```scala
def input(input: Input): State[Machine, (Int, Int)] =
  State { m =>
    val m1 = m.process(input)
    ((m1.coins, m1.candies), m1)
  }
```
<aside class="notes">
Refilling means adding new candies to the machine.
</aside>


### Refilling the machine
```scala
def refill(newCandies: Int): State[Machine, (Int, Int)] =
  State { m =>
    val m1 = m.copy(locked = true, newCandies + m.candies)
    ((m1.coins, m1.candies), m1)
  }
```
<aside class="notes">
Refilling means adding new candies to the machine.
</aside>


### Collecting
```scala
  def collect(): State[Machine, (Int, Int)] =
    State { m =>
      val m1 = m.copy(locked = true, coins = 0)
      ((m1.coins, m1.candies), m1)
    }
```
<aside class="notes">
Collecting means extracting the coins from the machine.
</aside>


### Combinator



### Combinator

>"A combinator is a higher-order function 
>that uses only function application and 
>earlier defined combinators 
>to define a result from its arguments."

Note: from https://en.wikipedia.org/wiki/Combinatory_logic


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
