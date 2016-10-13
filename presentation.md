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
* Modular
* Parallellizable

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

image src: https://en.wikipedia.org/wiki/Heraclitus#/media/File:Utrecht_Moreelse_Heraclite.JPG
</aside>

### Heraclitus
"All entities move and nothing remains still"


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


### Usage OO style

```scala
      val inputs: List[Input] = List(Coin, Turn, ...)

      val machine = Machine(locked = true, candies = 5, coins = 10)
      for (i <- inputs) machine process i
      machine.refill(100 - machine.candies)
      machine.collect()

      machine.coins shouldBe 0
      machine.candies shouldBe 100
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


### Combine in Maintenance


### Combine

```scala
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] = ???
     
        

  def flatMap[B](f: A => State[S, B]): State[S, B] = ???
        
            
    
```

<aside class="notes">
For combining State operations we need higher-order functions map and flatMap.
Map takes a function which takes an A and returns a B, which in turn returns a new State and B.
FlatMap
</aside>


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
