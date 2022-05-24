# Memory Management -- conflicts

1. Bignums

This only conflicts in a small area: for testing memory management, we have
added a built in function `test_refcount` that checks that the reference count
equals a number asked by the programmer.  It will have to be updated to work
with bignums instead of regular `i32`s.

Representative example program:
```python
class C(object):
  pass
x : C = None
x = C()

print(test_refcount(c, 1)) # Now 1 will be a pointer to a bignum instead of a regular i32
```
Most of the memory management tests use this built-in.

The fix will be to change the implementation `test_refcount`.

Additionally, we will need to implement a destructor for bignums. It will be
very simple since bignums contain no pointers internally.

2. Built-in libraries/Modules/FFI
3. Closures/first class/anonymous functions
4. Comprehensions
5. Destructuring assignment
6. Error reporting
7. Fancy calling conventions
8. For loops/iterators
9. Front-end user interface

For all of these features, there is no conflict!

They only change the frontend of the compiler, while the memory management only
changes the backend.

10. Generics and polymorphism

Generics are fully removed, turned into regular non-generic code, as part of
compilation, before the codegen and runtime.

Since memory management only changes the codegen and runtime, we do not conflict
with this feature.

11. I/O, files

This is implemented entirely in the front-end, with some built-ins at runtime,
so it should continue to work with memory management.

There is an opportunity to potentially do something cool here: the IO team wants
`close` to always be called on a file before it gets deleted, and we are adding
destructors that run automatically when something gets deleted. While we don't
currently plan on supporting custom destructors like this, it would be possible
to use this feature to automatically `close` files.

Test case:
```python
f : File = None
f = open(0)
print(test_refcount(f, 1)) # expected: True
```

12. Inheritance

Inheritance has a few interactions with memory management, mostly good ones. For
memory management, we are storing a table of destructors, and for inheritance,
they are storing a table of methods. We can merge these two tables, and save
space in the objects by storing just one table that contains both the destructor
and the methods.

To implement this improvement, only a little needs to change: we will change the
memory management code to use the methods table instead of our own `destructors`
table, and they will add the `$ClassName$$delete` functions as the first methods
of every class in the table.

We will also need to add entries for other objects like lists, strings, bignums,
and sets to the table.

Test case:
```python
class C(object):
    x: int = 4
    def sum(self) -> int:
        return x
class D(object):
    y: int = 12
    def sum(self) -> int:
        return x + y

x: C = None
x = D()
print(x.sum()) # expected: 9
print(test_refcount(x, 1)) # expected: True
```

13. Lists

The only concern for lists is that the memory layout will be different, with
added fields for memory management, so we have to make sure indexing still
works. But list operations compile to regular `load`/`store` IR operations, so
they will continue to work when integrated with memory management.

This test case tests that it still works:

```python
a: [int] = None
a = [1,2,3]
print(a[1]) # expected: 2
print(test_refcount(a, 1)) # expected: True
```

14. Optimization

Optimization has the potential to overlap, since we are both working on the IR,
but we talked to the optimization team and we don't think any of our changes
overlap: they are only working on the IR prior to codegen, while we only change
codegen.

Optimization could also impact us by e.g. removing dead variables, which could
change the reference counts and make our tests fail despite everything working
as it should. If our tests fail when merging with optimization, we will have to
look into it. Example:
```python
class C(object):
    pass
x : C = None
y : C = None # y is a dead variable
x = C()
y = x
print(test_refcount(x, 1)) # if DCE gets rid of y: expected: True
```

15. Sets and/or tuples and/or dictionaries

There are two conflicts here:
 - First, their hash set implementation is written in WASM, and assumes that
   there is no object header. So it will break when merged with memory
   management
 - Second, we will need to add a destructor for hash sets, like all the other
   types have.

16. Strings

Most of the string operations use the `load` and `store` from `memory.wat`, so
they will continue to work with memory management.

However, we will need to implement a destructor for strings. It will be very
simple, since they contain no pointers.

Test program:
```python
x : str = "abc"
print(test_refcount(x, 1)) # expected: True
```

