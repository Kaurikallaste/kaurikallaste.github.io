---
layout: post
title: "I Like Rust"
date: 2026-10-21 22:41
---

For my whole professional software developer career I have been writing PHP and TypeScript with a tiny bit of Java here and there. Recently I decided that I wanted to learn something new, so I picked up Rust. I have had fun, so I wanted to rant a little about PHP, its quirks, and how Rust handles similar situations.

## Static Types

This might seem basic, but coming from PHP this is the best thing since sliced bread. Yeah we have `declare(strict_types=1);` and with it you have static types ... kinda.

Magento's codebase for example is filled with code that looks something like this:
```php
/**
 * @param int $a
 * @return int
 */
function doSomething($a)
```

Is it type-safe? Hell no, I have no idea what it accepts or returns, just a pinky promise that it accepts an `int` and returns an `int`. Docblock types aren't enforced by anything, and without type declarations nothing is checked.

This is completely legit PHP code and it runs without any issues:
```php
<?php

declare(strict_types=1);

/**
 * @param int $a
 * @return int
 */
function doSomething($a)
{
    return "foobar";
}

var_dump(doSomething("23")); //string(6) "foobar"
```

This causes interesting scenarios like this:
```php
<?php

declare(strict_types=1);

/**
 * @param int $a
 * @return int
 */
function evilLegacyFunction($a)
{
    return "foobar";
}

function yourNotLegacyFunction(int $a): int
{
    if ($a > 5) {
        return $a;
    } else {
        return evilLegacyFunction($a);
    }
}


var_dump(yourNotLegacyFunction(5));
// PHP Fatal error:  Uncaught TypeError: yourNotLegacyFunction(): Return value must be of type int, string returned in code.php:19
```

In Rust you don't have this issue:
```rust
fn do_something(a: i32) -> i32 {
    a + 5
}
```

This will always accept an `i32` and return an `i32`. If it doesn't, the code won't compile, and the compiler will tell you where the error is.

## Structs

Arrays in PHP are truly a crime against humanity. They are hashmap-struct-arrays that do everything and are used for everything and it is practically impossible to enforce any type safety on them. Yes, I know, we have classes and we should use them, but a lot of code is still using this old pattern.

An `array` like this is probably the most common sight in most PHP codebases:
```php
$person = [
    'id' => 25,
    'name' => 'John',
    'age' => 50,
];
```

There are many issues with this:
* Nothing enforces types, you have to cast or check every time you read from here.
* Nothing enforces that the value at key actually exists: you might have `id`, you might not.
```php
$person2 = [
    'id' => 26,
    'name' => 'Jane',
]; // no age
```
* Even if you build them correctly they might still be later mutated to be wrong.
```php
$person['id'] = "1"; //legal
```

This forces you to write code that looks like this:
```php
if (!empty($person['age'])) { //check if age is set
    yourNotLegacyFunction((int) $person['age']); //cast age just to be safe
}
```

Yes, it's a solvable problem, but it's mental overhead that could be used to tackle the actual problem you are trying to fix.

In Rust you would just:
```rust
struct Person {
    id: u32,
    name: String,
    age: u32,
}

let person = Person {
    id: 25,
    name: String::from("John"),
    age: 50,
};
```

Every field is named and typed, everything is enforced, the values could never have the wrong type or be missing.

## Option Enum and Pattern Matching

How many times have you seen this error?
```php
Fatal error: Uncaught TypeError: someFunction(): Argument #1 ($value) must be of type int, null given, called in code.php on line 15 and defined in code.php:11
```

or this one?
```php
Warning: Undefined array key "foo" in code.php on line 11
```

or this?
```php
Fatal error: Uncaught TypeError: getSomeValue(): Return value must be of type int, null returned in code.php:6
```

Probably too many times, right? This bug cannot exist in Rust because it lacks the `null` type and has an enum called `Option` to represent values that could be absent. (Well, the bug _can_ exist, but using `unwrap` or `expect` is a choice you make, not an accident)

How it works in practice is something like this:
```rust
    let a = map.get("foo");

    match a {
        Some(value) => println!("{value}"),
        None => println!("didn't find value"),
    }

    // or

    if let Some(value) = map.get("foo") {
        println!("{value}");
    } else {
        println!("didn't find value");
    }
```

We `get()` a value from a `HashMap`. The `HashMap` get function returns an `Option<T>` enum that we have to match to either a) get the value or b) handle its absence. The compiler forces us to handle the sad path. No more bugs where you expected to have a value, and you always had a value (in testing), but for some reason, whenever Mercury is in retrograde, production crashes because of a null pointer exception that you forgot to handle.

This is probably the most common error in all of programming and it simply cannot exist, wonderful.

tl;dr compiler is fren

/rant
