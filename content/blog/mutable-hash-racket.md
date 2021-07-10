+++
title = "Implementing Simple Hash Tables in Racket"
date = 2021-01-01
description = "A simple implementation of mutable hash tables to learn how they work behind the scenes."
[taxonomies]
tags = ["racket", "data-structures"]
[extra]
katex = true
+++

[Hash Tables](https://en.wikipedia.org/wiki/Hash_table) are a fundamental data structure that most programmers will be familiar with. In this post, I will outline a simple implentation of hash tables in Racket, which will help us learn more about how they work, and how we can model these kinds of data structures in Racket. For this post, we will stick to *mutable* hash tables, though Racket does come with support for *immutable* ones, which would be tricker to implement.

## Hash Table Basics

Hash tables are a powerful data stucture since they allow us to index values by arbitrary keys while retaining (on *average*) $\text O(1)$ performance for lookups, insertion and deletion. They are able to acheive this by essentially acting as a layer on top of an array (in Racket, *vectors*). Vectors and hashes are similar, except vectors index their elements with integers, while hashes use keys.

To implement a hash table, then, our main goal is to create some *hash function* that converts our keys into integers that we can use to index a vector. When we *set* a value, we'll use the function to generate the insertion index, and when we *get* an item, we'll use the same function to find the index of the value to retrieve.

## The Hash Function

Hash functions can be quite complex in a practical implementation, but we will keep it simple by allowing only string keys, and using a simple conversion function. We want our function to take a string and generate some number to use for indexing. One way to do this is to sum the character codes of the characters in the string. In Racket:

```racket
(define (hash-func key)
  (for/sum ([char key]) (char->integer char)))
```
This generates a number, but we also want that number to be between 0 and the size of our table so that it can be used as an index. To do this, let's mod this value by the size of the table. We will also add two other nuances: We will start at a prime number, and multiply each char code by some other prime number. 

```racket
; Converts a string key to an integer in [0, size)
(define (hash-func key size)
  (let ([initial 17] [mult 13])
    (modulo 
      (+ initial
         (for/sum ([char key]) (* mult (char->integer char))))
      size)))
```

By using primes in hashing, we decrease the likelihood of *collisions*, the case in which two different keys produce the same index.[^1] We'll deal with collisions more later, but for now, this simple hash function will do.

## Sketching the API

Now that we have our hashing function in hand, let's remember what we are trying to create. Our goal in the end is to provide three basic functions for working with hashes:
- `(make-hash) -> hash?` : Creates a new, empty hash table
- `(hash-get hash key) -> any?` : Takes a key and returns the associated value
- `(hash-set! hash key value) -> void` : Sets the value for a specified key

To start, lets declare a struct which will serve as the basis for our design. For now, it's very simple, holding just one property: the table. We also specify the struct to be mutable so that the table property can be altered directly.

```racket
(struct hash (table) #:mutable)
```

By declaring the struct, Racket automatically provides us with a set of functions for constructing and accessing properties of the hash. Specifically `(hash name table)` creates a new hash, `(hash-table hash)` returns the table for a given hash, and `(set-hash-table! hash value)` sets a new value for the table.

With just this, we can create the first function in our API, `(make-hash)`, which constructs a new hash with an empty table. For now, we will create a table with a very small initial size, but in practice, this initial size would be much larger.

```racket
; Creates an empty hash table filled with the value null
(define (make-hash)
  (let ([size 3])
    (hash (build-vector size (λ (n) null)))))
```

Importantly, we must use a vector to hold the values in the table, since the performance of the data structure relies on being able to reference an index at an arbitrary position in constant time. If we were to use a linked list to hold the values in our table, this would not be possible.

We can now also make draft versions of our *get* and *set* functions. The get function takes a hash table and a key, and returns the value in the table associated with that key:

```racket
; Retrieve the value in the table associated with the key
(define (hash-get hash key)
  (let* ([table (hash-table hash)]
         [idx (hash-func key (vector-length table))])
    (vector-ref table idx)))
```

Simple enough; we can use the same logic to create a first draft of our set function:

```racket
; Set the value in the table for a given key
(define (hash-set! hash key value)
  (let* ([table (hash-table hash)]
         [idx (hash-func key (vector-length table))])
    (vector-set! table idx value)))
```

Perfect, let's test out these functions and see how they work:

```racket
> (define my-hash (make-hash))
> (hash-set! my-hash "key1" "value1")
> (hash-set! my-hash "key2" "value2")
> (hash-get my-hash "key1") ; --> "value1"
> (hash-get my-hash "key2") ; --> "value2"
```

So we can verify that this basic implementation works.

## Dealing With Collisions

Unfortunately, we're not done quite yet. The draft versions of the get and set functions we just wrote work, but they ignore the *collisions* we mentioned earlier. Take for example the keys "lies" and "foes". With our hash function, they both produce the same index value, since their character codes add up to the same value:

```racket
> (hash-func "lies" 3) ; --> 2
> (hash-func "foes" 3) ; --> 2
```

If we tried to store these keys in our hash table, the second value stored would overwrite our first value at the second index. There are several way to deal with collisions, but what we will do is instead of storing *just* the value at each position in the table, we will instead store a *list* of values. That way, if two keys collide, they will be put in a list together, rather than overwriting each other.

For this to work, we'll need to modify our get and set functions to act appropriately when there is a collision. Let's do the set function first. Recall that when we initiated our hash table, we filled it with the value `null`, which, in Racket, is the same as the empty list `'()`. So, all we have to do is `cons` our value onto the appropriate sublist, rather than setting it directly in the vector. If there already is a value at that position (i.e. a collision), we still just `cons` it to the list.

And, rather than just inserting the value, we will instead insert the key-value pair, so that way if there is a collision, we can select the appropriate value from the sublist when we *get* it later. With this change, we have this:

```racket
(define (hash-set! hash key value)
  (let* ([table (hash-table hash)]
         [idx (hash-func key (vector-length table))]
         [lookup (vector-ref table idx)])
    (vector-set! table idx (cons (cons key value) lookup))))
```

Now, if there is a collision, our table of values will look something like this, where the sublist at index 2 has two key-value pairs:

```racket
'(() () (("lies" . "value1") ("foes" . "value2")))`
```

We also need to update `hash-get` to pull out the correct value from these sublists, whether there is collision or not. To do this, once we narrow our search to the correct index in the table, we will use `findf` to search the sublist for the matching key-value pair[^2]. Then we'll return just the value, using `cdr`:

```racket
; Retrieve the value in the table associated with the key
(define (hash-get hash key)
  (let* ([table (hash-table hash)]
         [idx (hash-func key (vector-length table))])
    (cdr (findf (λ (pair) (equal? key (car pair)))
                (vector-ref table idx)))))
```

Note that `findf` performs a *sequential* search on the sublist, so if there are many collisions that land in the same position, our `hash-get` procedure could take up to linear time, $\text{O(n)}$, in the *worst* case. However, with a large enough table, and by using prime numbers in hashing, these collisions will be rare enough that our lookups will still run in constant time, $\text{O(1)}$, in the *average* case.

## Dynamic Hash Resizing

What we have so far works great, except there's one problem: we only allocate the size of our hash once, when we initiate it with `(make-hash)`. That means that as we fill up our hash with more values, we will start to get a lot more collisions, and with that, worse performance. To mitigate this, we can *dynamically* increase the size of the hash table when it starts to fill up.

One way to implement this is to keep track of a *load factor*, or the ratio between the number of values stored in the table and its size. When this load factor crosses a certain threshold, say 80% capacity, we will increase the size of our table.[^3]

In order to do this, let's edit our struct to also hold a counter for the number of items that are held in the table[^4]. This counter should start at 0 when a new hash is created, so let's give it an auto-value of 0:

```racket
(struct hash (table [num #:auto])
  #:mutable
  #:auto-value 0)
```

Then, in our `hash-set!` function, let's increment the counter[^5] and check if our load factor has reached the threshold. If it has, we resize the hash before setting the new value.

```racket
; Set the value in the table for a given key
(define (hash-set! hash key value)
  (set-hash-num! hash (add1 (hash-num hash)))
  (let ([load-factor (/ (hash-num hash) (vector-length (hash-table hash)))])
    (when (> load-factor .8)
      (hash-resize! hash)))
  (let* ([table (hash-table hash)]
         [idx (hash-func key (vector-length table))]
         [lookup (vector-ref table idx)])
    (vector-set! table idx (cons (cons key value) lookup))))
```
Now as we fill up the table and the load factor crosses 80%, the table will be automatically resized.

### hash-resize! 

You might have noticed we haven't actually written `hash-resize!` yet, so let's do that now. 

This function will first generate a new table that is twice the size of our old table. However, since we want to maintain the size of the table as a prime number, we will actually choose the size to be the *next prime* that occurs after doubling the original size. In the racket library `math/number-theory`, there is a handy function called `next-prime` that will help us here:

```racket
; Create a new table that is at least twice as large as the original.
(define (hash-resize! hash)
  (let* ([old-size (vector-length (hash-table hash))]
         [new-size (next-prime (* 2 old-size))]
         [new-table (build-vector new-size (λ (n) null))])
    (set-hash-table! hash new-table)))
```

We use `set-hash-table!` here since we want to completely swap our old table for this new one. 

We're not done though, since by doing this we've discarded the values in our old table. Instead we will need to *rehash* these values into the new table. This rehashing step is necessary since our hash function relies on the size of our table, so we can't reuse indices from the old table as they would not match up to their corresponding keys.

To rehash our values, we iterate over each sublist in the old table, and then over each key-value pair in those sublists. At each step, we want to calculate the new index using the hash function and insert into the new table.[^6] This can be acheived neatly with Racket's `for*` construct for nested loops:

```racket
; Create a new table that is at least twice as large as the original. 
; Values from the old table are rehashed into the new one.
(define (hash-resize! hash)
  (let* ([old-size (vector-length (hash-table hash))]
         [new-size (next-prime (* 2 old-size))]         
         [new-table (build-vector new-size (λ (n) null))])    
    (for* ([lst (hash-table hash)]           
           [pair lst]           
           #:unless (null? lst))       
      (let* ([idx (hash-func (car pair) new-size)]             
             [lookup (vector-ref new-table idx)])
        (vector-set! new-table idx (cons (cons (car pair) (cdr pair)) 
                                         lookup))))
    (set-hash-table! hash new-table)))
```

The `#:unless` guard lets us skip over null elements in the old table. Note that this resizing step, while helpful, is relatively expensive, as it requires a sequential walk through the original table. Additionally, `next-prime` can be expensive for large inputs. 

With our `hash-resize!` function written, our work is now complete! Now if we add to a table past 80% capactiy, it will automatically size up. If we set our inital size in the definition of `make-hash` to 3, then:

```racket
> (define my-table (make-hash))
> (hash-table my-table) ; --> '#(() () ())
> (vector-length (hash-table my-table)) ; --> 3 (initial size)
> (hash-set! my-table "key1" "value1")
> (hash-set! my-table "key2" "value2")
> (hash-set! my-table "key3" "value3") ; --> 
> (hash-table my-table) ; --> '#(() (("key3" . "value3")) (("key2" . "value2")) (("key1" . "value1")) () () ())
> (vector-length (hash-table my-table)) ; --> 7 (next prime after doubling 3)
```
## Conclusion

Hopefully with this exercise, you've gained some more intuition about how hash tables actually work behind the scenes. I also think this program can help show off some of the expressive power of Racket. If you have any questions or comments, please email me!

The full code can be found [here](https://gist.github.com/micahcantor/332b2952187402a01cb1aee8600949a9).

### References

This post is inspired by [Ben Awad's video](https://www.youtube.com/watch?v=UOxTMOCTEZk) on implementing hash tables in JavaScript. I also made use of [this article](https://www.interviewcake.com/concept/javascript/hash-map?) by InterviewCake that outlines the main concepts surrounding hash tables.

### Notes
[^1]: See [this answer on StackExchange](https://cs.stackexchange.com/questions/11029/why-is-it-best-to-use-a-prime-number-as-a-mod-in-a-hashing-function) for a more in depth explanation.

[^2]: If the same key has been set twice, it will collide, but the most recent key-value pair will appear first, so it will still be found first by findf. We could make this more precise by adding more logic to our set function to deal with this edge case.

[^3]: We could also dynamically decrease the size of the table as items are removed to decrease our memory footprint, but we won't get to that here.

[^4]: I don't love this solution, since it involves more mutation, but otherwise we would have to count the number of non-null elements in our table every time we wanted to calculate our load factor. This would need to be sequential, and so would force our insertions to occur in linear, rather than constant time, which we can't afford.

[^5]: This is inaccurate, since we shouldn't increment the counter when there is a collision, so there should be some extra logic here to check first.

[^6]: You'll notice this is the same code we used in hash-set! so we could pull out these lines into its own function instead of copy-pasting.