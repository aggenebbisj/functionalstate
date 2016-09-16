# Functional state

## From mutable to immutable

## Outline

- Why?
  - What was the problem?
  - How does immutability solve this? 
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

## Referential transparancy

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
trait RNG {	def nextInt: (RNG, Int)}
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
trait RNG {	def nextInt: (RNG, Int)}

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
def map[A,B](stateGenerator: RNG => (RNG, A))(f: A => B): RNG => (RNG, B) = rng => {  val (a, rng2) = stateGenerator(rng)    (f(a), rng2) // transform value here  }
  
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
def flatMap[A,B](f: Rand[A])(g: A => Rand[B]): Rand[B] = rng => {
  val (a, rngA) = f(rng)
  g(a)(rngA)
}
```


```                                                    
       -----                         -----         
       |   | -> rngA                 |   | -> rngB 
rng -> | f |             a -> rng -> | g |         
       |   | -> a                    |   | -> b
       -----                         -----         
```


```                                                    
       -----            -----         
       |   | -> rngA -> |   | -> rngB 
rng -> | f |            | g |         
       |   | -> a ----> |   | -> b
       -----            -----         
```

