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


### What is the problem?

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



## Referential transparancy


### Referential transparancy

>An expression is said to be  
referentially transparent  
if it can be **replaced**  
with its corresponding value   
**without changing** the program's behavior


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

## Introduction to state


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

## Combining state actions


### map2

```                        
                        -----
                        |   | ------------------> rngB
       -----            | B |            -----
       |   | -> rngA -> |   | -> b ----> |   |
rng -> | A |            -----            | f | -> c
       |   | -> a ---------------------> |   |
       -----                             -----
```


### sequence

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

