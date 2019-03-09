# Monads
## What is Monad
- A single unit
- A design pattern that allows structing programs generically while automating away boilerplate code needed by the program logic
- "裹在被子里打牌"
- wtf ???

# A world without Monad
We want to program only using functions.  

A pure function is a function where the return value is only determined by its input, without observable side effects. It means, a specific input will lead to a specific output.  

However, let's take random generator as an example:   

A random generator will access the random seed, which has "side effects": there're lots of states which are irrelavent with our code in these operations, and they are not explicit declared.  
  
`nextRandom` accepts no input but it will output different numbers, and it's relavent with time.   

```haskell
nextRandom :: Int -- Shouldn't it be a constant value?
```

In order to program it in a "functional" way, we need to write a function which explicit declares all the states in parameters. And we need to pass those states via parameters, and in the end we can get a specific output from a specific input.  

It's so complicated and hard to program.

# What is Monad again and Why Monad
We want to use functions which have "side effects".  

We don't want to handle all the states and environment by ourselves.  

Therefore, we can wrap an operation which has side effects and its states and values in a "package" and regard it as a new operation `Random Int`, where random is the package.  

```haskell
nextRandom :: Random Int
```

However, `Random Int` is not a `Int`, and `Random Int` + `Int` is illegal. We only care about pertinent values in the `Random` but not the `Random` itself, so how can we operate values in the `Random` without unwrap it? (If we unwrap it from `Random`, there will be states in our code again)  

Solution: compose them to construct a chain!  

We cannot do `Random Int` + `Int`, but we can do it in the `Random` by operate the pertinent values immediately after `nextRandom`.  

We have 
```haskell
nextRandom :: Random Int
plusOne :: Int -> Int
```

We use a `bind` (`>>=`) and a `return`:
```haskell
(>>=) :: Random Int -> (Int -> Random Int) -> Random Int
return :: Int -> Random Int

nextRandom >>= (return plusOne)
-- Actually it's unnecessary. We can use plusOne <$> nextRandom instead (functor). Monad can be used for something like "randomStringWithLength" to generate a random string with specific length.
```

Finally we get a `Random Int`!

# Who are Monads
- `Maybe` is Monad
- `IO` is Monad
- `List` is Monad
- `Random` is Monad
- `Either` is Monad
- `Try` is Monad
- `Reader` is Monad
- `Writer` is Monad
- `State` is Monad
- ......

# Maybe
Take 'Maybe' as an example.
```haskell
data Maybe a = Just a | Nothing
```

# Functor
'Maybe' is functor!
```haskell
-- fmap
(<$>) :: (a -> b) -> Maybe a -> Maybe b
f <$> (Just x) = Just (f x)
_ <$> Nothing = Nothing
```
So we have:
```haskell
f :: a -> b
(<$>) f :: Maybe a -> Maybe b
```

# Applicative
'Maybe' is applicative!
```haskell
-- inherit relation
Functor => Applicative

pure :: a -> Maybe a
pure = Just

-- apply
(<*>) :: Maybe (a -> b) -> Maybe a -> Maybe b
Nothing <*> _  = Nothing
(Just f) <*> x = fmap f x
```
So we have:
```haskell
x :: Num
f :: Maybe (a -> b)
pure x :: Maybe Num
apply f (pure x) :: Maybe Num
```

# Monad
'Maybe' is Monad!
```haskell
-- inherit relation
Applicative => Monad

-- No side effects are special cases with side effects
return :: a -> Maybe a
return x = Just x

-- bind
(>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
Nothing >>= _ = Nothing
(Just x) >>= f = f x

-- then
(>>) :: Maybe a -> Maybe b -> Maybe b
Nothing >> _ = Nothing
_ >> Nothing = Nothing
_ >> m = m

fail :: String -> Maybe a
fail _ = Nothing
```

# IO
```haskell
printW :: [String] -> IO ()
printW [] = return ()
printW (h:t) = putStrLn h >> printW t   
    
-- getLine :: IO String
-- words :: String -> [String]
f :: IO [String]
f = words <$> getLine

main :: IO ()
main = f >>= printW
```
With syntax sugar:
```haskell
printW :: [String] -> IO ()
printW [] = return ()
printW (h:t) = do
    putStrLn h
    printW t
    
f :: IO ()
f = do
    l <- getLine
    let w = words l
    printW w
```
