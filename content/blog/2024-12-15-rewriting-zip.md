+++
title = "Rewriting Rust's zip (poorly)"
[taxonomies]
  tags = ["rust"]
+++

Betwixt all of the Marvel Rivals and drumming I've been doing since I finished my fall semester, I thought it'd be a good idea to try rewriting `std::iter::zip`; in the past, I've followed along with Jon Gjengset's [Crust of Rust on Iterators](https://www.youtube.com/watch?app=desktop&v=yozQ9C69pNs&t=0s&ab_channel=JonGjengset) (to make `Flatten`) and tried my hand at `FlatMap` as well (see [Elton Pinto's blog](https://www.eltonpinto.me/blog/posts/implementing_flatmap_in_rust/) as a nice insight). This article is inspired by his, although will probably less involved as I found `zip` to be easier to implement, largely because `FlatMap`, for example, involves nested iterators and `FnMut` and other such fancy traits, while `Zip` is a more intuitive operation. There were only a few hiccups along the way.
\
\
\
As an aside, did you know that hiccough is a valid spelling, and hiccup is the more modernized version? English is dumb.

### What the hell is a zip?

Rust's `std::iter::zip` has one simple goal; when calling `zip` on two items that can be turned into iterators, we want `zip` to return pairs of each subsequent entry in each input! That's a mouthful, but a simple example from the docs will likely help:
```rust
let xs = [1, 2, 3];
let ys = [4, 5, 6];

let mut iter = zip(xs, ys);

assert_eq!(iter.next().unwrap(), (1, 4));
assert_eq!(iter.next().unwrap(), (2, 5));
assert_eq!(iter.next().unwrap(), (3, 6));
assert!(iter.next().is_none());
```

Now that you hopefully understand what's happening a bit better, the next step is

### Figuring out how to make this shit

Most things in the standard library are designed as follows:
1. Have a function to do a thing (`zip`), and create a struct that represents the thing you want the result to be `Zip`.
2. Use traits, associated types, and generics to implement functionality on top of said resulting thing `impl Iterator for ...`
3. Profit

So, let's get started with all of this, beginning with our `zip` function:
```rust
fn zip<A, B>(a: A, b: B) -> Zip<A::IntoIter, B::IntoIter>
where
    A: IntoIterator,
    B: IntoIterator,
{
    Zip::new(a, b)
}
```
We haven't quite implemented the `Zip` struct yet, but we will soon. For now, notice that intuitively we need both `a` and `b` to be able to be converted to Iterators; thus, we can place the `IntoIterator` ***trait*** bound on them. Having the bound be `Iterator` would be "too strict", in a sense, and would functionally require us to call `.into_iter()` on everything we'd pass in to `zip`.

Also, while `IntoIterator` is a trait, `IntoIter` is an ***associated type***! Associated types are type aliases associated with another type/trait. You may be thinking, this sounds a lot like generics, and yeah they're kind of similar, but the crucial difference is that associated types are specific to an implementation (i.e. in the `impl` block) of a trait, while generics allow multiple implementations. [This thread](https://www.reddit.com/r/rust/comments/waxk1l/what_is_the_difference_between_associated_types/) has a good explanation, which I'll attempt to summarize.

```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    // Required method
    fn into_iter(self) -> Self::IntoIter;
}
```
This is the definition of the `IntoIterator` trait, with the subsequent definition of the `IntoIter` type as well. Notice here that the type `IntoIter` resolves to a ***concrete type***, as IntoIterator is implemented on a specific type. For example, if we call `into_iter()` on an object of type `Vec<u32>`, then the `Item` is of type `u32`, and thus we create an `Iterator<Item = u32>`, which is exactly as desired.

Clearly, for each implementation of a given trait for a given type, we have a unique resolution for the associated type, which makes intuitive sense. On the flipside, something like `Add<Rhs>` can be implemented multiple times for a variety of types of `Rhs`, i.e. parametric polymorphism (generics).

Now that that's out of the way, let's get back to

### Zipping it! Time for structs

Our `zip` function wants an output of type `Zip<A::IntoIter, B::IntoIter>`, so let's do that.

```rust
pub struct Zip<A, B>
where
    A: IntoIterator,
    B: IntoIterator,
{
    a: A::IntoIter,
    b: B::IntoIter,
}
```

Well. That was easy.

The only thing we're missing is the `new` function that we used in `zip`, but that shouldn't be too hard:
```rust
impl<A, B> Zip<A, B>
where
    A: IntoIterator,
    B: IntoIterator,
{
    fn new(a: A, b: B) -> Zip<A::IntoIter, B::IntoIter> {
        Zip {
            a: a.into_iter(),
            b: b.into_iter(),
        }
    }
}
```
Notice how the return type matches our `zip` function signature; the rest is pretty reasonable.

### Implementing Iterators Interestingly (not)

The last thing we need to do is to make our `Zip` struct actually somewhat useful, so we need to implement the `Iterator` trait for it, which will get us all sorts of goodies for free (please do consult the [standard library](https://doc.rust-lang.org/std/iter/trait.Iterator.html) if you forget). The only thing we really need to implement for iterators is `next` (it's the "required method" they mention in the docs); here's the code:
```rust
impl<A, B> Iterator for Zip<A, B>
where
    A: Iterator,
    B: Iterator,
{
    type Item = (A::Item, B::Item);
    fn next(&mut self) -> Option<Self::Item> {
        let x = self.a.next()?;
        let y = self.b.next()?;
        Some((x, y))
    }
}
```

Here, we notice that `Item` is an associated type for `Iterator`; crucially, it's not predefined, and so we can set it to whatever we need. Here, since our goal is to get tuples of items from our two iterators, we simply do as described. The function implementation itself is also largely trivial -- just get the next values of both iterators, and return a tuple of them. If either is `None`, the `?` operator will gracefully handle this and return `None` for the overall iterator as desired.

### We're done??

Yes. 
\
\
\
\
\
\
\
\
Have a happy holidays and see you next time!
