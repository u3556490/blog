[The last time (and several times) around](./monads%20(extra%20time)%20-%20monads%20step%20by%20step.md) we tried to explain what a monad is in the mathematical sense, but until now we have seen little of its practical application, except the `Maybe` monad which specializes in exception handling. In this (hopefully) final episode we will go over the practicalities of certain types of monads, and during this journey, also somehow get a more complete picture of what functional programming is about.

> **Warning**: The following contains code snippets that are not guaranteed to work (because I'm too lazy to make them work). Nonetheless I hope you get the idea anyways. Read Learn You A Haskell for actually working code.

# Recap: What even is a monad anyway
A monad in programming is a data type that wraps up a piece of data in context and implements two functions:

-	`return :: a -> M a`
-	`bind :: M a -> (a -> M b) -> M b`

While making sure the functions obey the monad laws, i.e. `return` should be left- and right-identity to `bind` (notated `>>=` in Haskell), and `bind` should be associative. The `bind` function unwraps the data, applies a function to it, and since the function generates a new monad, returns this monad.

The monad is used to manage context for computations, such that repeating code tackling the context is separated out into the monadic `bind`. This way, the programmer can focus on the business process, while the nuances of composing computations are taken care of elsewhere.

In mathematics, a monad in some category $C$ is a monoid in the category of endofunctors in $C$, with functor composition as the monoidal binary operation. The monad laws stem from here and this stupid math provides a strong theoretical foundation for the programming monad.

# Baby steps with IO monad
In this section we look at a monad perhaps exclusive to Haskell: The IO monad, which wraps everything IO-related in a box. You may ask why this is required in the first place – that’s very true, let’s look at another property of functional programming for inspiration.

## Pure function
Functional programming is a programming paradigm that considers programs as pipelines, comprised of functions that do their own things composed together. This thought comes from mathematics and logic, and the programs being good enough to permit logical reasoning is important. For us, this basically means **we want programs that are easy to reason about** and debug. A crucial part of this is to minimize surprises by enforcing purity.

A function is **pure** if:
-	it is **idempotent**, i.e. same set of inputs give the same output.
-	It has **no side effects**, i.e. the output is everything the function ever does (the effect), and there are no other side behaviour.

For example, a random number generator (RNG) call is not idempotent, because every time you call that “`random()`” it gives you a different number. However, considering the whole procedure of initializing the RNG with some seed and then getting a number, it may be quite idempotent (returns the same RNG every time, if we fix the seed). In other words, anything that has a mutable internal state or refers to an external / global variable is not pure, since they can change regardless of our parameters (inputs). Thus, to write a good function, you may want to keep it idempotent by explicitly indicating all required information as parameters. An example of the opposite is as follows:

```python
my_variable = x

def my_impure(y):
    return x+y # impure: uses global variable
```

Since referring to mutable states and global variables are banned, that means we have eradicated much of the side effects. However, a function can have a console output step, which creates output (to the screen) not indicated in the function’s return values. This is a side effect, insofar as changing a global variable or internal state is a side effect. Side effects are like undocumented features: they can cause unwanted surprises. The following is a blatant example of side effects in a function:

```python
my_variable = x

def my_impure(y):
    print(y) # impure: conducts IO (side effect)
    x = x + 1 # impure: edits global variable
    return 0 # the “intended” effect here – always same output, but side effects render this impure
```

The corollaries of purity (if we want to – only some programming languages enforce it) are midly inconvenient, as a bunch of things now have to be explicitly stated in code. Variables should become immutable; and the code becomes somewhat more verbose. For instance, any IO operation changes the state of the monitor / files / hardware, so it must be given to and returned from the relevant functions. This adds clutter to the code and distracts programmers from the business logic, so we write a monad to isolate it. Thus, the IO monad is born.

## The IO monad
Haskell provides a variety of IO functions, all of which returning an instance of the IO monad. For example, to get a character you use the function `getChar :: IO Char` (returns `Char` wrapped inside an IO monad), and to print a character `putChar :: Char -> IO ()` (note the returned monad wraps nothing). The “empty” IO monad is not empty _per se_, because internally it represents the state of the world outside of the program, just that it doesn’t return a value for the program’s use this time.

Assuming you will always interact with IO one way or another as a part of a program, Haskell’s program entry point – `main` function – has type `IO ()`, i.e. does a bunch of things but always returns an empty IO monad. This way, the program you run can always **tap into the external situation somehow** (which is what IO in general is about) by using this IO monad in the context.

The IO monad by itself doesn’t do a lot, but you almost certainly would want to chain some IO-related and irrelevant operations together. Using `bind` we could do this:

```haskell
(some stuff)

-- recall: the >>= operator unwraps the IO monad, gets the value, and feeds it to the subsequent function
-- to make it bind properly, the return function is composed to it
(getChar >>= return . toUpperCase) >>= putChar

(some other stuff)
```

This can get out of hand if you want to refer to a prior value. This happens all the time, e.g. getting two values and passing them to a function. We can make use of the closure idea to expose the plain value unwrapped during the bind:

```haskell
getChar >>= (\value1 -> getChar >>= (\value2 -> doSomething value1 value2))

-- show the scope of variables better this way:
getChar >>= (\value1 ->
  getChar >>= (\value2 ->
    doSomething value1 value2))
```

`Bind` will apply the anonymous function we provide to the value inside the IO monad, but the anonymous function specifically allows the input be exposed to subsequent steps. In programming languages like Haskell (welp) there is something called “**do-notation**”, which provides the syntactic sugar to make such tricks readable:

```haskell
-- this is equivalent to the example above
do value1 <- getChar
   value2 <- getChar
   doSomething value1 value2

-- general syntax
do v <- f1
   f2
-- is the same as
f1 >>= (\v -> f2)
```

Note that the `do`-block is one expression in itself, and its return type and value are decided by the last line (last expression) inside the block. The `<-` operator actually indicates the end of the anonymous function and where the bind happens, and **may never be assumed to be an assignment operation**: the return type of `1` is a monad, it is not the same as that of `v`. The binding step may introduce “side effects” not visible to the `do`-block, and each monad has its own implementation of `bind`. 

Not introducing a `v <-` on a line means the unwrapped value of this line’s operation is “discarded”, i.e. not available for future steps.

```haskell
do f1
   f2

-- is the same as
f1 >>= (\_ -> f2)
-- Haskell allows this “then” operator as well
f1 >> f2
```

The do-notation provides an easy way to think about the business logic (without caring about monad syntax). For instance, a bunch of `Maybe` binds can be simplified and the unhappy path handling becoming transparent to us. Without syntactic sugar like this, it would be imaginably quite annoying to concoct a user-interactive program like below (from HaskellWiki):

```haskell
nameDo :: IO ()
nameDo = do putStr "What is your first name? "
            first <- getLine
            putStr "And your last name? "
            last <- getLine
            let full = first ++ " " ++ last
            putStrLn ("Pleased to meet you, " ++ full ++ "!")

main = nameDo
```

Doesn't this look like an imperative program? It does - but never forget this is syntactic sugar hiding the hideous `bind` operations, which actually dictate the program's behaviour.

The IO monad provides a light introduction to the further uses of monads. Notably, the IO monad itself isn’t created via actually pure code, but let’s forget about it for now.

# Writer monad: monoid redux
There are many places you can create a monad actually. Consider this situation: 

We are writing a program and this program needs to maintain a log trail for each action (yes, the Learn You A Haskell example). Because we can't keep a global log variable, each individual function (action) may look like this:

```haskell
greaterThan :: Int -> Int -> (Bool, String)
greaterThan x y
  | x > y = (True, "Bigger")
  | x < y = (False, "Smaller eh")
  | otherwise = (False, "Equal eh")

addFive :: Int -> (Int, String)
addFive x = (x+5, "Added five")

subSix :: Int -> (Int, String)
subSix x = (x-6, "Removed six")

-- putting them together somehow
action :: Int -> (Bool, String)
action x = let (x', log) = addFive x
               (x'', log') = subSix x'
               (y, log'') = greaterThan x' x''
           in (y, log ++ log' ++ log'')           
```

When we chain the functions together, we pass the computed values around, and finally concatenate the logs as one. The final answer and log are returned together. Over time we may start seeing a pattern, especially in the function type signatures:

```haskell
greaterThan :: Int -> Int -> (Bool, String)
addFive ::            Int -> (Int, String)
subSix::              Int -> (Int, String)
action ::             Int -> (Bool, String)

-- in general...
someFunction :: a -> a -> (b, c)
someOthers ::        a -> (b, c)
```

The above assumes our actual computation is with some types `a` and `b`, while the log is being accumulated in another type `c`; we can get more flexibility later on. The `a -> (b, c)` portion is pervasive, and we might be able to generalize the function composition logic shown above.

```haskell
-- "w@(v, log)" means we pattern-match the structure called "w" to find "(v, log)" inside. Just a reminder this is one tuple.
chain :: (a, String) -> (a -> (b, String)) -> (b, String)
chain w@(v, log) f = let (v', log') = f v in (v', log ++ log')
```

And if we want to generalize the "`String`" part, i.e. instead of keeping a textual log and accumulate whatever (e.g. keep a total token count if you're calling OpenAI APIs over several function calls), we need to think carefully what to do. With strings we just concatenate them no problem, any function not doing logs can just provide an empty string and our programs won't crash. Likewise, we can use arrays to replace strings here.

Does this ring a bell? Yes, we're looking for **monoids**. Monoids are the data types that support naive aggregation like this well. Haskell, specifically, requires a data type to come with two things to qualify as a monoid:

- `mappend`: standing for *monoid append*, this is the binary operation that puts two monoidal values together. For strings this is the concatenation operation.
- `mempty`: standing for *empty* element of the *monoid*, this defines the identity element. For strings this is the empty string.

Now we can rewrite the `chain` function like this:

```haskell
-- the notation "(Monoid m) => m" means we require the type "m" to be a monoid.
-- also forgive me for using "ax" to stand for "accumulator", I know this isn't a register.
chain :: (Monoid m) => (a, m) -> (a -> (b, m)) -> (b, m)
chain (v, ax) f = let (v', ax') = f v in (v', mappend ax ax')
```

And now we can chain whatever we like together. Haskell goes one step further and anticipates this sort of activity, where we want to **write to a shared resource that may be passed from one processor (function) to the next**, by providing the Writer monad. If this sounds a bit forced to you, remember how there are repeating patterns like `a -> (b, c)` in the code:

```haskell
-- we begin with this data type the functions always return. we need to put them together
-- this is not valid syntax because we must use concrete types instead of just "a" and "m"...
type Result = (a, m)

-- the function that generate "Result" is, in its simplest form, this:
operation :: a -> Result 

-- creating a new "Result" in the stupidest way also fits this pattern
create :: a -> Result
create x = (x, mempty)

-- the chaining step
-- note the type of the value wrapped inside "Result" may differ, we'll get to that later
chain :: Result -> (a -> Result) -> Result
```

You may notice this essentially describes a monad:

- `create` = `return` because it wraps a minimal piece of data in the context
- `chain` = `bind` because, well, the type signature gave it away - it is a method to compose actions on the `Result` data type
  - the actions themselves give a new `Result`, so this is not a simple functor!

The Haskell implementation looks like this:

```haskell
-- the declaration of the type: parametrized over monoid w and value a, it wraps a tuple.
data Writer w a = Writer (a, w)

-- this line below says a Writer containing a monoidal w is a monad.
instance (Monoid w) => Monad (Writer w) where
  -- the "return" function
  return x = Writer (x, mempty)
  
  -- the "bind" function as infix operator
  (Writer (x, m)) >>= f = let (Writer (y, m')) = f x in (Writer (y, mappend m m'))
```

Then the composition of several functions is now cleaner:

```haskell
import Control.Monad.Writer

addFive :: Int -> Writer String Int
addFive x = Writer (x+5, "Added five")

subSix :: Int -> Writer String Int
subSix x = Writer (x-6, "Removed six")

action :: Int -> Writer String Int
action x = do
    x' <- addFive x
    x'' <- subSix x'
    return (x' + x'')  -- example: you can get a value plus all the logs!

main = putStrLn $ show $ action 5
```

Next, we will look at the exact opposite of this scenario: reading from a global variable.

# Reader monad and functions as monads
The Writer monad we saw just now was probably not hard to wrap your head around - it is merely a wrapper for a tuple, so log-taking effects are contained. To do the opposite, i.e. **reading from some global variable that accompanies a computation**, is going to be much more sophisticated. Before we dive into this realm possibly so full of mustard gas, take this with you as a gas mask: "Functions are computations, and can encode computations that happen later."

Based on the Writer pattern, we can think of the contrasting "Reader" pattern like this: Every time we want to do something, we must provide in the parameters the global variable to read, and this parameter is pretty much never changed. By explicitly supplying the global / environment variable upon which the function will depend, we can achieve what is known as **dependency injection** in OOP parlance. A naive way to write and compose functions to do so is as follows.

```haskell
-- "env" stands for the data type of our global environment variable
someFunction :: a -> env -> b
otherFunction :: b -> env -> c

-- the hard part is to bring them together. for the two functions above, we can do this:
result :: c
result = let v1 = someFunction x e
             v2 = otherFunction v1 e
         in v2

-- in general, just composing one value and one function
chain :: a -> env -> (a -> env -> b) ->  b
chain x e f = f e x
```

An observation we can make here is that we are composing functions of arity two. If we try to curry the signature `a -> env -> b` we get `a -> (env -> b)`, which is an obvious way of showing we can supply the environment variable later upon invocation - this makes it even more like dependency injection. In fact, we can supply the computation to make in terms of `env -> b` (a function), and thus effectively delay the computation for later until we do provide the details of what to do.

However, the code will get much messier and harder to reason about - what even does "a function that takes in a global variable and spits out an output" mean? With a bit of type kungfu we can see something interesting hidden in `chain`.

```haskell
-- if we curry the function supplied it can be like this
chain' :: a -> env -> (a -> (env -> b)) -> b
-- or perhaps, by swapping the order of the inputs...
chain'' :: env -> a -> (a -> (env -> b)) -> b
-- you may notice there is a repeated pattern (env -> x) here
chain_v3 :: (env -> a) -> (a -> (env -> b)) -> b
```

The key step is that we rebracketed the input in the form of a "*function*" (`env -> a`). It is based on the idea that we can delay the evaluation of our computations by specifying the details later: By turning the input into a function, we do not know the `env` and the `a` upon composing two actions with `chain_v3`. To get the value `a` for the subsequent computation (`a -> (env -> b)`), we need to execute the prior function (`env -> a`) first; this in turn requires the environment variable, ultimately fetched somehow from the start of the computation chain. The construct `env -> a` indicates "given this certain environmental setup, what will the outcome of the computation be?"

Letting the type gymnastics run amok, we can do this:

```haskell
-- continuing where we left off
-- evaluating the prior computation can provide the input to the subsequent
chain_v3 :: (env -> a) -> (a -> (env -> b)) -> b
chain_v3 prev next = let v = executeSomehow prev
                         f' = next v  -- gives a function, you see?
                     in executeSomehow f' 

-- actually, if we keep the output in the same format, we can chain things further
-- (now this returns a function!)
chain_v4 :: (env -> a) -> (a -> (env -> b)) -> (env -> b)
chain_v4 prev next = next (executeSomehow prev)
-- to get the value, we supply the environment variable
answer = (chain_v4 (chain_v4 f1 f2) f3) e
```

The way how this works is we are building a line of dominoes without triggering it. Stating this once again: When we chain `f1` with `f2`, we say the effect of `f1` should pass into `f2`, but we can't evaluate that now - the first domino `f1` needs the global variable as a push. By the time we finished chaining the computations we want, we go back and give the push, so the whole evaluation finally happens. We'll see in a bit how to make this more useful for programmers.

If the type signature of `chain_v4` looks familiar to you, yes, there is a monad lying in wait: Haskell calls this design pattern the "`Reader`" monad. It is defined like this:

```haskell
-- type declaration
newtype Reader env a = Reader { runReader :: env -> a }
-- this syntax's equivalent statement:
-- Reader wraps only one thing, a function
data Reader env a = Reader (env -> a)
-- with another function to evaluate this wrapped function
runReader :: Reader env a -> env -> a
runReader (Reader f) x = f x

-- Reader is a functor
-- remarks: (1) the map function is called "fmap" (2) the dollar sign ($) means "open a bracket here and close it at the end of the line"
instance Functor (Reader env) where
  fmap f (Reader g) = Reader $ f . g

-- the monad definition
instance Monad (Reader env) where
  -- barebones Reader to wrap a minimal piece of context: regardless of the environment, give x
  return x = Reader $ \_ -> x

  -- bind
  (Reader f) >>= g = Reader $ \x -> runReader (g (f x)) x
```

That's a lot to digest! One simpler way to think about this is to **consider the function wrapped inside `Reader` your actual computation** work to do. Let's break it down part by part.

1. The type declaration  
The syntax is different - it is a one-liner to declare a wrapper for a function, and a "getter" that executes the function inside. The `runReader` function is the entry point that kickstarts the domino chain and commences the evaluation.

2. `Reader` is a functor  
Here, it is important to note how applying a function to the wrapped function inside the `Reader` structure is just composing them together outright.

3. The `return` function  
To introduce something to the `Reader` context minimally, simply use a function that always returns that something. Remember to wrap it up inside a `Reader` box.

4. The `bind` function  
This is the only complicated part. Taking the syntax apart, you can see we first use some environment / read-only value `x` to evaluate the prior function `f`, then the result is fed to the subsequent function `g`. The result from `g` (we call `r`) is a `Reader` and it's almost perfect - the only problem is we need that `x` to begin with. Therefore, we next create an anonymous function taking the `x` and running the `r :: Reader`. Finally, we wrap it inside a `Reader` for the sake of the type signature.

```haskell
-- warning: not real syntax! the first "x" is undefined.
(Reader f) >>= g = let v = f x
                       r = g v
                   in Reader (\x -> runReader r x)
```

> We can alternatively have `m@(Reader f) >>= g = Reader $ \x -> runReader (g (runReader m x)) x` and `fmap f m@(Reader g) = Reader $ \x ->  f (runReader m x)`. Note that we are not pattern-matching into the monads supplied in this alternative implementation, so we need to `runReader` to get correct chainings.

Haskell also has a `MonadReader` typeclass, which provides additional qualify-of-life benefits, and the `Reader` we have is of course an instance of that. The goodies include:

```haskell
instance MonadReader (Reader env) where
  -- note: same as Reader $ \x -> x
  ask = Reader id

  local :: (env -> env) -> (Reader env) -> (Reader env)
  local f (Reader g) = Reader $ \x -> runReader g (f x)
  -- alternative implementation
  local f m  = Reader $ runReader m . f
```

The `ask` function is used inside `Reader` context to expose the environment variable value. When composed with `bind`, it spits out any environment variable supplied upon evaluation as it is, so the functions down the line can take it (e.g. in do-notation). The `local` function is used so we can run a portion of our code inside a modified environment. The function `f` changes the environment specified, before we run it.

Interestingly, because of how `bind` works, it is possible to opt-out from using the environment variable - unless you `ask`, you won't get it in context.

With all that done, we can look at an example (from [StackOverflow](https://stackoverflow.com/questions/77224093/reader-monad-in-haskell-where-is-the-reader-passed-as-argument)) to see the `Reader` monad in action:

```haskell
import Control.Monad.Reader

tom :: Reader String String
tom = do
    env <- ask
    return (env ++ " This is Tom.")

jerry :: Reader String String
jerry = do
  env <- ask
  return (env ++ " This is Jerry.")

tomAndJerry :: Reader String String
tomAndJerry = do
    t <- tom
    j <- jerry
    return (t ++ "\n" ++ j)

runJerryRun :: String
runJerryRun = (runReader tomAndJerry) "Who is this?"

main = putStrLn runJerryRun
```

If this has been confusing to you, here's some good news: The Reader monad is almost entirely unnecessary apart from marking our intent, it is at best a mere wrapper of a function. Can we use functions directly without the man-in-the-middle? The answer is yes, **functions are monads**.

```haskell
-- defines stuff for functions typed (a -> whatever).
-- note: we simply deleted the "Reader"-related bits.
instance Functor ((->) a) where
  fmap = (.)

instance Monad ((->) a) where
  return x = \_ -> x
  f >>= g = \x -> g (f x) x

instance MonadReader ((->) a) where
  ask = id
  local f g = g . f
```

It is that simple - you can easily understand that. In fact, you may notice how this is quite similar to the `chain_v4` back from the start. With essentially the same syntax, we can do the same example again:

```haskell
-- home-made "ask" function
ask :: String -> String
ask s = s

tom :: String -> String
tom = do
    env <- ask
    return (env ++ " This is Tom.")

jerry :: String -> String
jerry = do
  env <- ask
  return (env ++ " This is Jerry.")

tomAndJerry :: String -> String
tomAndJerry = do
    t <- tom
    j <- jerry
    return (t ++ "\n" ++ j)

main = putStrLn $ tomAndJerry "Who is this?"
```

After all this longwinded wangjangling, we have explained the Reader monad. There remains the crucial takeaway that functions (and types wrapping them) can encode an action to be completed later, as demonstrated in the Reader monad, where we needed to `runReader` before things finally happen. Such phenomenon occurs almost entirely because we chose to curry the function `a -> env -> b` as `a -> (env -> b)`; many monads are just fancy function currying. In any case, this paves the way to the final objective: the State monad.

# The stately State monad
The best way to explain the State monad is to combine the Writer and Reader in one context. We have seen how the Writer allows writing to a (monoidal) piece of "global" data and the Reader permits read-only access to a shared resource, both implemented as monads to satisfy pure functional programming constraints. **A piece of data that is meant to be both readable and writable to our computations can thus be modelled as Writer plus Reader** rolled homogenously as one.

This provides the obvious intuition for the State monad. Let's look at its implementation piecemeal:

```haskell
newtype State s a = State { runState :: s -> (a, s) }
-- reminder: runState looks like this
runState :: State s a -> s -> (a, s)
runState (State f) x = f x
```

The State monad is parametrized over two data types, `a` being the type of our computation and `s` our state. As a two-tuple, you can imagine how the Writer logic will be applicable: we chain computations on `a` which is exposed to subsequent functions, and somehow "accumulate" the changes to the state in `s`. The monad wraps a function, which alludes to Reader logic: given some initial state of type `s`,  what will the result of the computation and final state be? As before, this function represents our stateful computation to be made.

Next we define the monadic functions:

```haskell
instance Monad (State s) where
  return x = State $ \s -> (x, s)
  (State f) >>= g = State $ \s -> 
    let (v, s') = f s
        (State h) = g v
    in h s'
```

The `return` function is straightforward: given a piece of data `x`, put it inside a minimal `State`, where regardless of the situation we're in the computed result remains `x`.

The `bind` function is different: using the same approach as Reader, we first evaluate the older stateful computation `f` using some state `s` (which we will obtain later), then its result `v` is passed to the subsequent function `g`. This returns a new `State` containing a stateful computation `h`. Instead of returning it directly, we connect it to `s` through the anonymous function, so it will be possible to properly retrieve the result of `h`. An alternative implementation can be like so:

```haskell
-- this might be even more comprehensible.
f >>= g = State $ \s -> 
  let (v, s') = runState f s
  in runState (g v) s'
```

> Interestingly, we do not specifically need to restrict any type to be a monoid, because we allow arbitrary computations on `s` and `a` instead of always using `mappend` on either one.
>
> Another curiosum is whenever we have a monad wrapping two types (e.g. `State s a`), we only declare part of it as a monad (e.g. `State s`). That means, we can let the other type do whatever it wants, but as long as `s` is fixed in one type, we consider it the same monad type anyways.

The "API" to work with the state value `s` inside the State monad consists of, most notably, these two functions:

```haskell
get :: State s s
get = State $ \s -> (s, s)

put :: s -> State s ()
put s = State $ \s -> ((), s)
```

The `get` function sets the result value to whatever state we current have. Inside the context (e.g. with binds and do-notation), using `get` exposes the state to further manipulation, not just the computation part. The opposite is achieved using `put`, which sets to the given state and does not provide a computation result. In a `do` block this replaces the current state.

Learn You a Haskell has a simple example of the State monad being used to implement a stack:

```haskell
-- this will not work out of the box. to make it work, use "import Control.Monad.State as State",
-- and replace all occurrences of "State" type hint with "State.State", "State" constructor with "State.state".
-- this trick may be required for all code examples. 
import Control.Monad.State

type Stack = [Int]

-- based on get and put, we can design our stack pop and push functions
-- (x:xs) is a pattern that matches to an array, x being the first element and xs being the rest. it can also be a constructor
pop :: State Stack Int
pop = State $ \(x:xs) -> (x, xs)
push :: Int ->  State Stack ()
push x = State $ \s -> ((), x:s)

stackyStack :: State Stack Int 
stackyStack = do
    push 2
    push 1
    stackNow <- get
    if stackNow == [1,2,3]
        then put [8,3,1]
        else put [9,2,1]
    v <- pop
    return v

-- fst means "get the first element of". "fst $ runState" is also a function called "evalState"
runIt :: Int
runIt = fst $ runState stackyStack [3]

-- show is the "toString" function.
main = putStrLn $ show runIt
```

Whoops, the State monad introduction looks a bit anticlimactic - we spent much more time building the basics with Writer and Reader.

# Wrapping up the episode
## Further perils you may feel free to get yourself into
From here on out there shouldn't be too many things about monad that will severely puzzle you. Yes, I mean there *are*, and you better watch out for things like:

- Continuation monad  
Continuation passing is a style of programming where you always (or just oftentimes) provide a callback when you call a function, so when it's done it will call that callback. This may occur when one wants to add flexibility in the code: Let's say we have a function of type `a -> b` and change it to `a -> c -> b` so a parameter can be supplied later, before we curry the `c` and get `a -> (c -> b)` (i.e. moving the `c` to the right hand side of the "`=`" sign in a Haskell function definition). The composition of similar functions generate a pattern captured by the continuation monad, which works quite like the monads we have seen today.

- MonadPlus  
A monad is a monoid in the category of endofunctors, but when we also consider the type they wrap, there is no guarantee that monoid-like behaviour is everywhere and free to use. That is, a type `a` can be a monoid when provided the suitable operation and identity, but the monads `m a` may not. Haskell has this "`MonadPlus`" thingy that introduces a binary operation `mplus :: m a -> m a -> m a` and an identity element `mzero` to define, so the monadic type is always a monoid as well.

- Monad transformers  
This happens when you want to wrap a monad in another to get the behaviour of both, e.g. an `IO` wrapping a `Maybe`. Monad transformers allow this wrapping and result in a new monad, with its own `bind` function to take care of composition boilerplate.

## Takeaways
> *"Holy Shit!"* - Learn You a Haskell Community Version

We've learnt a lot today:

- Pure functions are easy to reason about because they are idempotent and have no side effects.
- The IO monad is a pattern to capture IO activities for pure functional programming.
- The Writer monad is a pattern for doing computations and accumulating some progress / result indicator simultaneously. This allows writing to a shared global variable.
- The Reader monad models computations as functions that evaluate later, when an environment variable is supplied. This allows reading from a shared global variable.
- Functions are monads themselves.
- The State monad is a pattern to permit the use of a shared global state, which you can read from and write to at will.

That's all, folks. If you want more you may pick up where I refuse to continue: reading [the original monad paper](https://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf).