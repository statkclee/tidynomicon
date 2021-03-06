---
title: "Non-Standard Evaluation"
output: md_document
permalink: /non-standard-evaluation/
questions:
  - "When and how does R evaluate code?"
  - "How can we take advantage of this?"
objectives:
  - "Trace the order of evaluation in function calls."
  - "Explain what environments and expressions are and how they relate to one another."
  - "Justify the author's use of ASCII art in the second decade of the 21st Century."
keypoints:
  - "R uses lazy evaluation: expressions are evaluated when their values are needed, not before."
  - "Use `expr` to create an expression without evaluating it."
  - "Use `eval` to evaluate an expression in the context of some data."
  - "Use `enquo` to create a quosure containing an unevaluated expression and its environment."
  - "Use `quo_get_expr` to get the expression out of a quosure."
  - "Use `!!` to splice the expression in a quosure into a function call."
---



The biggest difference between R and Python is not where R starts counting,
but its use of [lazy evaluation](../glossary/#lazy-evaluation).
Nothing truly makes sense in R until we understand how this works.

## How does Python evaluate function calls?

Let's start by looking at a small Python program and its output:


```python
def ones_func(ones_arg):
    return ones_arg + " ones"
def tens_func(tens_arg):
    return ones_func(tens_arg + " tens")
initial = "start"
final = tens_func(initial + " more")
print(final)
#> start more tens ones
```

When we call `tens_func` we pass it `initial + " more"`;
since `initial` has just been assigned `"start"`,
that's the same as calling `tens_func` with `"start more"`.
`tens_func` then calls `ones_func` with `"start more tens"`,
and `ones_func` returns `"start more tens ones"`.
But there's a lot more going on here than first meets the eye.
Let's spell out the steps:


```python
def ones_func(ones_arg):
    ones_temp_1 = ones_arg + " ones"
    return ones_temp_1
def tens_func(tens_arg):
    tens_temp_1 = tens_arg + " tens"
    tens_temp_2 = ones_func(tens_temp_1)
    return tens_temp_2
initial = "start"
global_temp_1 = initial + " more"
final = tens_func(global_temp_1)
print(final)
```

Step 1: we assign `"start"` to `initial` at the global level:

```
           +--------
    global | initial       ----> value:"start"
           +--------
```

Step 2: we ask Python to call `tens_func(initial + "more")`,
so it creates a temporary variable to hold the result of the concatenation
*before* calling `tens_func`:

```
           +--------
           | global_temp_1 ----> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Step 3: Python creates a new stack frame to hold the call to `tens_func`:

```
           +--------
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Note that `tens_arg` points to the same thing in memory as `global_temp_1`,
since Python passes everything by reference.

Step 4: we ask Python to call `ones_func(tens_arg + " tens")`,
so it creates another temporary:

```
           +--------
           | tens_temp_1   ----> value:"start more tens"
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Step 5: Python creates a new stack frame to hold the call to `ones_func`:

```
           +--------
 ones_func | ones_arg      --+
           +--------         |
           | tens_temp_1   --+-> value:"start more tens"
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Step 6: Python creates a temporary to hold `ones_arg + 3`:

```
           +--------
           | ones_temp_1   ----> value:"start more tens ones"
 ones_func | ones_arg      --+
           +--------         |
           | tens_temp_1   --+-> value:"start more tens"
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Step 7: Python returns from `ones_func`
and puts its result in yet another temporary variable in `tens_func`:

```
           +--------
           | tens_temp_2   ----> value:"start more tens ones"
           | tens_temp_1   ----> value:"start more tens"
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

Step 8: Python returns from `tens_func`
and puts its result in `final`:

```
           +--------
           | final         ----> value:"start more tens ones"
           | global_temp_1 ----> value:"start more"
    global | initial       ----> value:"start"
           +--------
```

The most important thing here is not that I'm using a million lines of code on a supercomputer to draw ASCII art,
but rather the fact that Python evaluates expressions *before* it calls functions,
and passes the results of those evaluations to the functions.
This is called [eager evaluation](../glossary/#eager-evaluation),
and is what most modern programming languages do.

## How does R evaluate the same kind of thing?

R,
on the other hand,
uses [lazy evaluation](../glossary/#lazy-evaluation).
Here's an R program that's roughly equivalent to the Python shown above:


```r
ones_func <- function(ones_arg) {
  paste(ones_arg, "ones")
}

tens_func <- function(tens_arg) {
  ones_func(paste(tens_arg, "tens"))
}

initial <- "start"
final <- tens_func(paste(initial, "more"))
print(final)
#> [1] "start more tens ones"
```

And here it is with the intermediate steps spelled out in a syntax I just made up:


```r
ones_func <- function(ones_arg) {
  ones_arg.RESOLVE(@tens_func@, paste(tens_arg, "tens"), "start more tens")
  ones_temp_1 <- paste(ones_arg, "ones")
  return(ones_temp_1)
}

tens_func <- function(tens_arg) {
  tens_arg.RESOLVE(@global@, paste(initial, "more"), "start more")
  tens_temp_1 <- PROMISE(@tens_func@, paste(tens_arg, "tens"), ____)
  tens_temp_2 <- ones_func(paste(tens_temp_1))
  return(tens_temp_2)
}

initial <- "start"
global_temp_1 <- PROMISE(@global@, paste(initial, "more"), ____)
final <- tens_func(global_temp_1)
print(final)
```

While the original code looked much like our Python,
the evaluation trace is very different,
and hinges on the fact that
*an expression in a programming language can be represented as a data structure*.

> **What's an Expression?**
>
> An expression is anything that has a value.
> The simplest expressions are literal values like the number 1,
> the string `"stuff"`, and the Boolean `TRUE`.
> A variable like `least` is also an expression:
> its value is whatever the variable currently refers to.
>
> Complex expressions are built out of simpler expressions:
> `1 + 2` is an expression that uses `+` to combine 1 and 2,
> while the expression `c(10, 20, 30)` uses the function `c`
> to create a vector out of the values 10, 20, 30.
> Expressions are often drawn as trees like this:
>
>         +
>        / \
>       1   2
>
> When Python (or R, or any other language) reads a program,
> it parses the text and builds trees like the one shown above
> to represent what the program is supposed to do.
> Processing that data structure to find its value
> is called [evaluating](../glossary/#evaluation) the expression.
>
> Most modern languages allow us to build trees ourselves,
> either by concatenating strings to create program text
> and then asking the language to parse the result:
>
>     left <- '1'
>     right <- '2'
>     op <- '+'
>     combined <- paste(left, op, right)
>     tree <- parse(text = combined)
>
> or by calling functions.
> The function-based approach is safer and more flexible,
> so once we introduce the way R handles regular function calls,
> we'll dive into that.

Step 1: we assign "start" to `initial` in the [global environment](../glossary/#global-environment):

```
           +--------
    global | initial       ----> value:"start"
           +--------
```

Step 2: we ask R to call `tens_func(initial + "more")`,
so it creates a [promise](../glossary/#promise) to hold:

-   the [environment](../glossary/#environment) we're in (which I'm surrounding with `@`),
-   the expression we're passing to the function, and
-   the value of that expression (which I'm showing as `____`, since it's initially empty).

```
           +--------
           | global_temp_1 ----> PROMISE(@global@, paste(initial, "more"), ____)
    global | initial       ----> value:"start"
           +--------
```

and passes that into `tens_func`:

```
           +--------
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), ____)
    global | initial       ----> value:"start"
           +--------
```

Crucially,
the promise in `tens_func` remembers that it was created in the global environment:
it's eventually going to need a value for `initial`,
so it needs to know where to look to find the right one.

Step 3: since the very next thing we ask for is `paste(tens_arg, "tens")`,
R needs a value for `tens_arg`.
To get it,
R evaluates the promise that `tens_arg` refers to:

```
           +--------
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

This evaluation happens *after* `tens_func` has been called,
not before as in Python,
which is why this scheme is called "lazy" evaluation.
Once a promise has been resolved,
R uses its value,
and that value never changes.

Steps 4:
`tens_func` wants to call `ones_func`,
so R creates another promise to record what's being passed into `ones_func`:

```
           +--------
           | tens_temp_1   ----> PROMISE(@tens_func@, paste(tens_arg, "tens"), ____)
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

Step 5:
R calls `ones_func`,
binding the newly-created promise to `ones_arg` as it does so:

```
           +--------
 ones_func | ones_arg      --+
           +--------         |
           | tens_temp_1   --+-> PROMISE(@tens_func@, paste(tens_arg, "tens"), ____)
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

Step 6:
R needs a value for `ones_arg` to pass to `paste`,
so it resolves the promise:

```
           +--------
 ones_func | ones_arg      --+
           +--------         |
           | tens_temp_1   --+-> PROMISE(@tens_func@, paste(tens_arg, "tens"), "start more tens")
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

Step 7: `ones_func` uses `paste` to concatenate strings:

```
           +--------
           | ones_temp_1   ----> value:"start more tens ones"
 ones_func | ones_arg      --+
           +--------         |
           | tens_temp_1   --+-> PROMISE(@tens_func@, paste(tens_arg, "tens"), "start more tens")
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

Step 8: `ones_func` returns:

```
           +--------
           | tens_temp_2   ----> "start more tens ones"
           | tens_temp_1   --+-> PROMISE(@tens_func@, paste(tens_arg, "tens"), "start more tens")
 tens_func | tens_arg      --+
           +--------         |
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

Step 9: `tens_func` returns:

```
           +--------
           | final         ----> "start more tens ones"
           | global_temp_1 --+-> PROMISE(@global@, paste(initial, "more"), "start more")
    global | initial       ----> value:"start"
           +--------
```

We got the same answer,
but in a significantly different way.
Each time we passed something into a function,
R created a promise to record what it was and where it came from,
and then resolved the promise when the value was needed.
R *always* does this:
if we call:


```r
sign(2)
```

then behind the scenes,
R is creating a promise and passing it to `sign`,
where it is automatically resolved to get the number 2 when its value is needed.
(If I wanted to be thorough,
I would have shown the promises passed into `paste` at each stage of execution above,
but that's a lot of typing even for me.)

## Why is lazy evaluation useful?

R's lazy evaluation seems pointless
if it always produces the same answer as Python's eager evaluation,
but as you may already have guessed,
it doesn't have to.
To see how powerful lazy evaluation can be,
let's create an expression of our own:


```r
my_expr <- expr(a)
```

Displaying the value of `my_expr` isn't very exciting:


```r
my_expr
#> a
```

but what kind of thing is it?


```r
typeof(my_expr)
#> [1] "symbol"
```

A symbol is a kind of expression.
It is not a string (though strings can be converted to symbols and symbols to strings)
nor is it a value---not yet.
If we try to get the value it refers to, R displays an error message:


```r
eval(my_expr)
#> Error in eval(my_expr): object 'a' not found
```

We haven't created a variable called `my_expr`,
so R cannot evaluate an expression that asks for it.

But what if we create such a variable now and then re-evaluate the expression?


```r
a <- "this is a"
eval(my_expr)
#> [1] "this is a"
```

More usefully,
what if we create something that has a value for `a`:


```r
my_data <- tribble(
  ~a, ~b,
  1,  10,
  2,  20
)
my_data
#> # A tibble: 2 x 2
#>       a     b
#>   <dbl> <dbl>
#> 1     1    10
#> 2     2    20
```

and then ask R to evaluate our expression in the **context** of that tibble:


```r
eval(my_expr, my_data)
#> [1] 1 2
```

When we do this,
`eval` looks for definitions of variables in the data structure we've given it---in this case,
in the tibble `my_data`.
Since that tibble has a column called `a`,
`eval(my_expr, my_data)` gives us that column.

This may not seem life-changing yet,
but being able to pass expressions around
and evaluate them in various contexts allows us to seem very clever indeed.
For example,
let's create another expression:


```r
add_a_b <- expr(a + b)
typeof(add_a_b)
#> [1] "language"
```

The type of `add_a_b` is `language` rather than `symbol` because it contains more than just a symbol,
but it's still an expression,
so we can evaluate it in the context of our data frame:


```r
eval(add_a_b, my_data)
#> [1] 11 22
```

Still not convinced?
Have a look at this function:


```r
run_many_checks <- function(data, ...) {
  conditions <- list(...)
  checks <- vector("list", length(conditions))
  for (i in seq_along(conditions)) {
    checks[[i]] <- eval(conditions[[i]], data)
  }
  checks
}
```

It takes a tibble and some logical expressions,
evaluates each expression in turn,
and returns a vector of results:


```r
run_many_checks(my_data, expr(0 < a), expr(a < b))
#> [[1]]
#> [1] TRUE TRUE
#> 
#> [[2]]
#> [1] TRUE TRUE
```

We can take it one step further and simply report whether the checks passed or not:


```r
run_all_checks <- function(data, ...) {
  conditions <- list(...)
  checks <- vector("logical", length(conditions))
  for (i in seq_along(conditions)) {
    checks[[i]] <- all(eval(conditions[[i]], data))
  }
  all(checks)
}

run_all_checks(my_data, expr(0 < a), expr(a < b))
#> [1] TRUE
```

Just to make sure it's actually working, we'll try something that ought to fail:


```r
run_all_checks(my_data, expr(b < 0))
#> [1] FALSE
```

This is cool, but typing `expr(...)` over and over is kind of clumsy.
It also seems superfluous,
since we know that arguments aren't evaluated before they're passed into functions.
Can we get rid of this and write something that does this?


```r
check_all(my_data, 0 < a, a < b)
```

The answer is going to be "yes",
but it's going to take a bit of work.

> **Square Brackets... Why'd It Have to Be Square Brackets?**
>
> Before we go there,
> a word (or code snippet) of warning.
> The first version of `run_many_checks` essentially did this:
>
> 
> ```r
> conditions <- list(expr(a + b))
> eval(conditions[1], my_data)
> #> [[1]]
> #> a + b
> ```
>
> What I did wrong was use `[` instead of `[[`,
> which meant that `conditions[1]` was not an expression---it wa a list containing a single expression:
>
> 
> ```r
> conditions[1]
> #> [[1]]
> #> a + b
> ```
>
> It turns out that evaluating a list containing an expression produces a list of expressions rather than an error,
> which is so helpful that it only took me an hour to figure out my mistake.

## What is tidy evaluation?

Our goal is to write something that looks like it belongs in the tidyverse.
We'll start by creating a tibble to play with:


```r
both_hands <- tribble(
  ~left, ~right,
  1,     10,
  2,     20
)
both_hands
#> # A tibble: 2 x 2
#>    left right
#>   <dbl> <dbl>
#> 1     1    10
#> 2     2    20
```

We want to be able to write this:


```r
check_all(both_hands, 0 < left, left < right)
```

without calling `expr` to quote our expressions explicitly.
For simplicity's sake,
our first attempt only handles a single expression:


```r
check_naive <- function(data, test) {
  eval(test, data)
}
```

When we try it, it fails:


```r
check_naive(both_hands, left != right)
#> Error in eval(test, data): object 'left' not found
```

This makes sense:
by the time we get to the `eval` call,
`test` refers to a promise that represents the value of `left != right` in the global environment.
Promises are not expressions---each promise contains an expression,
but it also contains an environment and a copy of the expression's value (if it has ever been calculated).
As a result,
when R sees the call to `eval` inside `check_naive` it automatically tries to resolve the promise that contains `left != right`,
and fails because there are no variables with those names in the global environment.

So how can we get the expression out of the promise without triggering evaluation?
One way is to use a function called `substitute`:


```r
check_using_substitute <- function(data, test) {
  subst_test <- substitute(test)
  eval(subst_test, data)
}
check_using_substitute(both_hands, left != right)
#> [1] TRUE TRUE
```

However,
`substitute` is frowned upon because it does one thing when called interactively on the command line
and something else when called inside a function.
Instead,
we should use a function called `enquo` which returns an object called a [quosure](../glossary/#quosure)
that contains only an unevaluated expression and an environment.
Let's try using that:



```r
check_using_enquo <- function(data, test) {
  q_test <- enquo(test)
  eval(q_test, data)
}
check_using_enquo(both_hands, left != right)
#> <quosure>
#> expr: ^left != right
#> env:  0x7f8ee8f12c88
```

Ah: a quosure is a structured object,
so evaluating it just gives it back to us in the same way that evaluating `2` or `"hello"` would.
What we want to `eval` is the expression inside the quosure,
which we can get using `quo_get_expr`:


```r
check_using_quo_get_expr <- function(data, test) {
  q_test <- enquo(test)
  eval(quo_get_expr(q_test), data)
}
check_using_quo_get_expr(list(left = 1, right = 2), left != right)
#> [1] TRUE
```

All right:
we're ready to write `check_all`.
As a reminder,
our test data looks like this:


```r
both_hands
#> # A tibble: 2 x 2
#>    left right
#>   <dbl> <dbl>
#> 1     1    10
#> 2     2    20
```

Our first attempt (which only handles a single test) is a deliberate failure:


```r
check_without_quoting_test <- function(data, test) {
  data %>% transmute(result = test) %>% pull(result) %>% all()
}
check_without_quoting_test(both_hands, left < right)
#> Error in mutate_impl(.data, dots): object 'left' not found
```

Good:
we expected that to fail because we're not enquoting the test.
(If this *had* worked, it would have told us that we still don't understand what we're doing.)
Let's modify it to enquote and then pass in the expression:


```r
check_without_quoting_test <- function(data, test) {
  q_test <- enquo(test)
  x_test <- quo_get_expr(q_test)
  data %>% transmute(result = x_test) %>% pull(result) %>% all()
}
check_without_quoting_test(both_hands, left < right)
#> Error in mutate_impl(.data, dots): Column `result` is of unsupported type quoted call
```

Damn---we thought this one had a chance.
The problem is that when we say `result = x_test`,
what actually gets passed into `transmute` is a promise containing an expression.
Somehow,
we need to prevent R from doing that promise wrapping.

This brings us to `enquo`'s partner `!!`,
which we can use to splice the expression in a quosure into a function call.
`!!` is pronounced "bang bang" or "oh hell",
depending on how your day is going.
It only works in contexts like function calls where R is automatically quoting things for us,
but if we use it then,
it does exactly what we want:


```r
check_using_bangbang <- function(data, test) {
  q_test <- enquo(test)
  data %>% transmute(result = !!q_test) %>% pull(result) %>% all()
}
check_using_bangbang(both_hands, left < right)
#> [1] TRUE
```

We are almost in a state of grace.
The two rules we must follow are:

1.  Use `enquo` to enquote every argument that contains an unevaluated expression.
2.  Use `!!` when passing each of those arguments into a tidyverse function.


```r
check_all <- function(data, ...) {
  tests <- enquos(...)
  result <- TRUE
  for (t in tests) {
    result <- result && (data %>% transmute(result = !!t) %>% pull(result) %>% all())
  }
  result
}

check_all(both_hands, 0 < left, left < right)
#> [1] TRUE
```

And just to make sure that it fails when it's supposed to:


```r
check_all(both_hands, left > right)
#> [1] FALSE
```

Backing up a bit,
`!!` works because [there are two broad categories of functions in R](https://tidyeval.tidyverse.org/getting-up-to-speed.html#whats-special-about-quoting-functions):
[evaluating functions](../glossary/#evaluating-function) and
[quoting functions](../glossary/#quoting-function).
Evaluating functions take arguments as values---they're what most of us are used to working with.
Quoting functions, on the other hand, aren't passed the values of expressions, but the expressions themselves.
When we write `both_hands$left`, the `$` function is being passed `both_hands` and the quoted expression `left`.
This is why we can't use variables as field names with `$`:


```r
the_string_left <- "left"
both_hands$the_string_left
#> Warning: Unknown or uninitialised column: 'the_string_left'.
#> NULL
```

The square bracket operators `[` and `[[`, on the other hand,
are evaluating functions,
so we can give them a variable containing a column name and get either a single-column tibble:


```r
both_hands[the_string_left]     # single square brackets
#> # A tibble: 2 x 1
#>    left
#>   <dbl>
#> 1     1
#> 2     2
```

or a naked vector:


```r
both_hands[[the_string_left]]   # double square brackets
#> [1] 1 2
```

## What have we learned?

Delayed evaluation and quoting are confusing for two reasons:

1.  They expose machinery that most programmers have never had to deal with before (and might not even have known existed).
    It's rather like learning to drive an automatic transmission and then switching to a manual one---all of a sudden
    you have to worry about a gear shift and a clutch.
2.  R's built-in tools don't behave as consistently as they could,
    and the functions provided by the tidverse as alternatives use variations on a small number of names:
    `quo`, `quote`, and `enquo` might all appear on the same page.

If you would like to know more,
or check that what you now think you understand is accurate,
[this tutorial][lyttle-tutorial] by Ian Lyttle is a good next step.
