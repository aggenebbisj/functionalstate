## Functional state

---

## Referential transparancy

* Testable
* Composable
* Modular
* Parallellizable

---

```scala
def rollDie: Int = {
    val rng = new scala.util.Random
    rng.nextInt(6)
}
```