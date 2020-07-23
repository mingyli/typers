# RAII [WIP]
RAII stands for Resource Allocation Is Initialization—yes that term makes no sense. In reality, it’s an incredibly powerful tool to handle the incredibly common problem of resource management. It’s also one of the foundational building block principles of Rust.

# The Resource Management Problem

Assume that we have a resource `R` with the following pattern:

- In order to use the resource `R`, you must first call `open(R)`. Any operations on an unopened resource results in an error.
- After you have done using resource `R`, you should call `close(R)`. Not doing so would result in a resource leak.
- It is an error to use `R` after it has been closed
- It is an error to close an already closed resource.

This should seem reasonable, albeit a little bit abstract. Let’s focus on a concrete example that implements the resource pattern—the Python file API:

- `open(filename, mode)` returns a file object that you can then manipulate
- You can call `read()` or `write()` on `f` to read and write data
- After using the file, you should call `f.close()`. All operations on a closed file result in errors.

I chose Python because it’s basically pseudocode (xD). Let’s assume for a second that Python doesn’t have a garbage collector, and not calling `close()` would result in a memory leak. (This is actually a reasonable assumption, as more complex resources may need more cleaning up work to do than just freeing up memory.) Let’s write a simple program:

    def write_asdf_to_file(filename):
      f = f.open(filename)
      f.write("asdf")
      f.close()

Seems pretty straightforward right? But resource management quickly becomes cumbersome as we increase the complexity of the program:

    def write_contents_to_file(contents, filename):
      f = f.open(filename)
      if !check_contents(contents):
        return
      final_contents = process_contents(contents)
      f.write(final_contents)
      f.close()

This code would actually result in a resource leak! If `check_contents(..)` returns `False`, we forget to close the file! This brings us to the crux of the resource management problem—manual resource management is ugly, easy to forget, and makes code super clunky. There’s also something else we didn’t take into account—exceptions! What if `write()` throws an exception? What if open throws an exception? What if we had two files instead of one? What if one open fails and the other succeeds? Our code quickly becomes extremely complex and hard to reason about—even though functionally it is pretty simple.

    def transfer_contents(file1, file2):
      try:
        f1 = open(file1)
      except IOError as e:
        return
    
      try:
        f2 = open(file2)
      except IOError as e:
        f1.close()
        return
    
      # Both f1, f2 are guarenteed to be open.
      contents = f1.read()
      if !check_contents(contents):
        f1.close()
        f2.close()
        return
    
      f2.write(process_contents(contents))
    
      f1.close()
      f2.close()

What a nightmare. We didn’t even handle the case where `read()` and `write()` might throw exceptions, but that is an exercise for the reader. (This is also a case for why `try` statements suck, but that’s a discussion for another time).

# The Solution—RAII

The more Python inclined would probably be internally screaming—use `with`! Don’t worry, we will get there. 
It’s clear that manual resource management, even with a set of such simple requirements, can make code incredibly complex. RAII is an ingenious solution to this: use **scope** to manage resources attached to a variable. When you construct the variable, call `open()` on the resource (hence the name Resource Allocation IS Initialization). When the variable goes out of scope, call `close()`. Your mind should be blown right now. By tying resource management to scope, we have reduced a complex problem to a very simple one for both you and the compiler to reason about. Programmers reason about scope all the time, and with RAII, knowledge about scope means they have knowledge about resource management, for free!

- If a variable is initialized, you can **always** perform operations on it. Sure, you might forget to initialize a variable, *but the compiler can check that!*
- If a variable goes out of scope, it is closed **automatically**. No more manual closes and additional cleanup cluttering up your code.
- It is **impossible** to perform operations on a closed resource because it is out of scope.

Let’s rewrite that clunky, incomplete `transfer_contents()` function from earlier, with an envisioned RAII file API (if Python had complete RAII support):

    def transfer_contents(file1, file2):
      f1 = File(file1)
      f2 = File(file2)
    
      contents = f1.read()
      if !check_contents(contents):
        return # No more annoying cleanup!
    
      f2.write(process_contents(contents))

Take a moment to appreciate the simplicity. 
The other extremely appealing aspect about RAII is *how broadly it applies*. Almost everything can be thought of as a resource.

- Files, db connections, sockets, web connections—traditional resources that no longer need to be manually managed. Most of these have a pretty natural `open()` and `close()` concepts.
- Locks. This was another mind blower for me. `open() = lock_acquire()` and `close() = lock_release()`. Imagine coding an OS without ever having to even think about having to release a lock. These objects are typically called lock guards. Here is pseudo-pythonic lock guard API based on C++’s lock guard API:
    lock = Lock()
    def write_contents_to_file_multithreaded(filename, contents):
      guard = lock_guard(lock) # Constructing a guard calls lock.acquire().
      file = File(filename)
    
      if !check_contents(contents)
        return # guard goes out of scope, lock.release() called automatically.
    
      file.write(process_contents(contents))
- Memory. That’s right. `open() = new [C++] = malloc [C]` and `close() = delete [C++] = free [C]`. RAII memory management lets us have our cake and eat it too. We can get all the performance advantages of a language without garbage collection and never have to worry about memory leaks! 
# Language Support for RAII
## Partial Support—Python, Java, C#

Because RAII is such a crazy good idea, a lot of languages have partial support. All these languages have “block level RAII” (I made this term up) that can be used to manage a multitude of resources, with the exception of memory. Let’s see an example of block level RAII with our favorite demo language Python. 

    def write_contents_to_file_with_real_python(filename, contents):
      with open(filename) as f: # Create a scope with f declared. Call f.__enter__()
        if !check_contents(contents):
          return
        final_contents = process_contents(contents)
        f.write(final_contents)
      # f is out of scope. f.__exit()__ called.

The way `with` block works is that when code enters the block `f.__enter__()` is called, and when the code exits the block `f.__exit__()` is automatically called. You can think of this as just syntactic sugar on top of the pure RAII principles described above. The `with` acts as the constructor, and `__enter__()` as as `open()`. When the scope created by `with` is exited, the variable is out of scope and `__exit__()` is called, the equivalent of `__drop__()`. 
RAII in C# and Java are implemented in a pretty similar way—with something called a `try-with-resources` statement. While the syntax is slightly different, the principles are the same. They declare a block scope where RAII is active for some sort of variable.

## RAII Alternatives—Go


## Full RAII Support—C++
    int i = 0;
    std::mutex mu_;
    
    void my_function() {
      std::lock_guard lock(mu_);
      i += 1;
    }
    
    class lock_guard {
      lock_guard(std::mutex&) { acquire the mutex }
      ~lock_guard() { release the mutex }
    }
## The Crown Jewel—Rust

