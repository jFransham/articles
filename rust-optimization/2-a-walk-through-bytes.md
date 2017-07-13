# A walk through `bytes`

If you read [the first article in this series on optimization in Rust][
opt-part-one], you may remember that I made a significant point on improving
cache locality and data structures. However, although I benchmarked the CPU
cycle-shaving optimizations, I didn't dive quite as deeply into memory layout.
So, let's do that now.

The crate I'm going to focus on today is `bytes`, a highly-optimized replacement
for `Vec<u8>`/`Arc<Vec<u8>>` that includes a bunch of smart tweaks that we'll
get into today. The most notable as a user of this library is that it has
copy-on-write semantics - the buffer is reference-counted but when you want to
mutate it you can get a uniquely-owned version of your own.  As you'll see
though, that's far from the only trick up its sleeve.

I will note that although the first article required a strong knowledge of Rust,
this one may require more in-depth knowledge. If you don't understand anything
here, you may want to look up some information on unsafe Rust and come back
later. If you don't know where to look, you can ask me in the issues on this
repo. A good source of information is [What every programmer should know about
memory][memory-article].

[memory-article]: https://lwn.net/Articles/250967/

So with that said, let's check out what this struct looks like!

```rust
struct Bytes {
    inner: Inner2,
}
```

Oh, well, that's not very interesting. Turns out the actual functionality of
`Bytes` is split up among a couple of other structs. This is because there are
two structs, `Bytes` and `BytesMut`, which are identical except that functions
implemented on the latter can allow some extra functionality and ignore some
runtime checks since it's impossible for a user of the library to construct a
`BytesMut` that doesn't uniquely own its data. The actual runtime representation
of `Bytes` and `BytesMut` is identical, however.

-- TODO: Move this to a "lessons learned" section
In general, having a seperate struct that represents an "already checked"
version of another struct is a great pattern, and one that can be added in all
kinds of programs. For example, you could have a `Sorted<T>` type that's just
represented by a `Vec<T>` or `Box<[T]>` under the hood but implements
`From<...>` in a way that sorts its data before returning the new type. Then,
when a function receives data of type `Sorted<T>` it statically knows that this
data is sorted already.

Here's the definition of `Inner2`:

```rust
struct Inner2 {
    inner: Inner,
}
```

...and the definition of `Inner` (we'll get to why we have two layers here
later):

```rust
#[cfg(target_endian = "little")]
#[repr(C)]
struct Inner {
    arc: AtomicPtr<Shared>,
    ptr: *mut u8,
    len: usize,
    cap: usize,
}

#[cfg(target_endian = "big")]
#[repr(C)]
struct Inner {
    ptr: *mut u8,
    len: usize,
    cap: usize,
    arc: AtomicPtr<Shared>,
}
```

Let's talk about endianness. Integers are represented internally to CPUs as a
series of bytes. Essentially they're base-256 numbers - a series of digits, each
of which can have 256 different values. If you're reading this article then you
understand English and you're probably a programmer, so you'll understand the
concept of base 10 and base 16 (hexadecimal). The way we traditionally write
numbers is big-endian, with the largest number first. For example, in the number
123 the number 1 represents the largest power of 10 (100), the number 2 is
the next-largest power of 10 (10) and the 3 is the smallest power of 10 (1).
Big-endian architectures have the first byte represent the largest power of 256,
the next byte the second-largest power of 256, etc. Little-endian architectures
have the _smallest_ power first, so the number we would traditionally write as
"123" would be written "321" in little-endian decimal. Likewise, the number
"66051" is represented as `[0x00, 0x01, 0x02, 0x03]` in big-endian and
`[0x03, 0x02, 0x01, 0x00]` in little-endian. The bits within the bytes are
always in big-endian order.

So, why do we care? Usually we don't, but `bytes` has a smart optimization. We
know that `Inner` - and therefore, `Bytes` - will always be the size
`4 * mem::size_of::<usize>()` (assuming that
`mem::align_of::<usize>() <= mem::size_of::<usize>()`, which is a reasonable
assumption). So, if we make one of the pointers a "tag" (just like the tags used
for `enum`s) we can use the rest of the struct to store a small number of bytes.
We can say, `if arc == 0`, we can use `ptr`, `len` and `cap` to store the byte
array. Since we know that 0 is an invalid pointer, this is safe.

The eagle-eyed among you might have noticed that alignment doesn't matter here,
since in both definitions above `ptr`, `len` and `cap` form a contiguous block
and can be used as a byte array. That's because `Bytes` doesn't use `arc` as a
tag, it uses _the least-significant 2 bits_ of `arc` as a tag. We know the
alignment of `Shared` is at least 4 (it's actually 4 on 32-bit architectures and
8 on 64-bit architectures) and so any pointer to it must be a multiple of 4.
Therefore the bit representing 1 and the bit representing 2 will always be 0. We
can then say (munged a bit for succinctness):

```rust
#[cfg(target_endian = "little")]
const INLINE_OFFSET: isize = 1;
#[cfg(target_endian = "big")]
const INLINE_OFFSET: isize = 0;

unsafe fn get_pointer(val: *const Inner) -> *const u8 {
  (val as *const u8).offset(INLINE_OFFSET)
}

const KIND_MASK: usize = 0b11;
const KIND_INLINE: usize = 0b01;

// Bitwise inversion of KIND_MASK, truncated to 1 byte size
const INLINE_LEN_MASK: usize = 0b11111100;
// The number of bits taken up by the tag
const INLINE_LEN_SHIFT: usize = 2;

let inner = Inner { ptr: 0 as _, len: 0, cap: 0, arc: 0 as _ };
let arc = inner.arc as usize;
let kind = arc & KIND_MASK;

if kind == KIND_INLINE {
  let len = (arc & INLINE_LEN_MASK) >> INLINE_LEN_SHIFT;

  let slice = slice::from_raw_parts(get_pointer(&inner as *const _), len);

  // {...}
}
```

That is, we can store extra data in the unused bytes of `arc`. Not only this,
but since we don't use the entirety of the least-significant byte of `arc` for
the tag we can store _even more data_ in the unused bits. Here, we use it for
the inline length, but we'll see later that we can play other tricks with it
too. You can imagine that if we had an `uN` type for arbitrary values of N that
it would look like this:

```rust
#[cfg(target_endian = "little")]
struct InnerWithInlineBytes {
  length: u6,
  tag: u2,
  data: [u8; mem::size_of::<Inner>() - 1],
}

#[cfg(target_endian = "big")]
struct InnerWithInlineBytes {
  data: [u8; mem::size_of::<Inner>() - 1],
  length: u6,
  tag: u2,
}
```

So, `KIND_INLINE` is the first of our kind tags. Let's meet the whole gang:

```rust
const KIND_ARC: usize = 0b00;
const KIND_INLINE: usize = 0b01;
const KIND_STATIC: usize = 0b10;
const KIND_VEC: usize = 0b11;
const KIND_MASK: usize = 0b11;
```

As promised, the one that causes `arc` to be a valid pointer (`KIND_ARC`) has
the tag set to 0. This means that no masking is necessary to extract the
pointer. We'll get to that one last, since it's the most complex. For now, I'm
going to burn through `KIND_STATIC` quickly (because it's not very interesting)
and then explain `KIND_VEC`.

So, `KIND_STATIC` just means that the `ptr` and `len` point to a
`&'static [u8]`. This means that when a `Bytes` is being created from a static
buffer it doesn't need to be cloned until it's written to, unlike a `Vec` which
always requires cloning, even when being created from a static buffer. It would
be possible to allow the `Bytes` struct to refer to a slice of any lifetime
`'a`, but not without sacrificing the current API. There's an [issue filed about
this][lifetime-generic-bytes] already. It's worth noting that no other language
in wide use today allows you to do this. If you wrote an equivalent API in C++,
for example, you couldn't write an API that prevents users from passing in a
pointer to a byte array that is allocated on the stack. A lot of C++ code clones
its input in cases like this, just to be safe. Here, we can enforce that any
slice we're pointing to will never be deallocated.

[lifetime-generic-bytes]: https://github.com/carllerche/bytes/issues/119

So, `KIND_VEC`. This, like `KIND_STATIC`, is largely to prevent cloning when
creating a `Bytes` directly from a `KIND_VEC`. There's actually a [PR that has
yet to be merged at time of writing][into-vec-pr] that also prevents cloning
when converting back, so `Vec::from(Bytes::from(some_vec))` is more-or-less
zero-cost.

[into-vec-pr][https://github.com/carllerche/bytes/pull/151]

With this kind, `ptr`, `len` and `cap` are essentially the same as you get in
`Vec`. In fact, the implementation of this kind just instantiates a vector,
takes a note of its pointer, length and capacity and then `mem::forget`s it.
When it wants to drop the memory it simply uses `Vec::from_raw_parts` and allows
`Vec`'s `Drop` implementation take care of the rest. If you don't know how `Vec`
works, the [section in `Vec`'s documentation regarding allocation and guarantees
][capacity-and-reallocation] is a good place to start. If you want more
information, you can check out a deep dive into `Vec`'s internals in the
[nomicon][nomicon].

[capacity-and-reallocation]: https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation
[nomicon]: https://doc.rust-lang.org/nomicon/vec.html

So, here's the main event: `KIND_ARC`. This is a slice with shared ownership,
similar to `Arc<[u8]>`. The layout of `Inner` now looks like this:

```rust
struct Inner {
    /// The pointer to the shared slice data. We'll get to what this looks like
    /// in a moment
    full_slice_pointer: AtomicPtr<Shared>,
    /// Pointer to the start of our slice. We'll explain why this is important
    /// in a moment
    start_of_slice: *mut u8,
    /// Length of our slice
    length_of_slice: usize,
    /// No longer used for `KIND_ARC` (we do still care about capacity, but we
    /// store it elsewhere)
    _capacity: usize,
}
```

The _actual_ implementation of the data still looks the same as the `Inner`
struct defined before, this is just a more descriptively-named version for the
sake of explanation.

`AtomicPtr<T>` is the same in memory as `*mut T`, except that operations to set
and get the pointer are atomic - you can write and read it from different
threads without the risk of reading a partially-written value. This allows our
`Bytes` values to be shared across threads safely. Our `Shared` struct is very
simple, and is essentially the same as `Arc<Vec<T>>` (or, rather, [the inner
value that `Arc` is a pointer to][arc-inner]). The only reason to not use
`Arc<Vec<T>>` is because `AtomicPtr<Shared>` is easier to work with when you're
doing trickly stuff with the lower 2 bits of the pointer. You can do the same
with `Arc`, but it involves messy transmutes and reimplementing the `Arc` logic
is simpler. Also, `Arc` counts the number of weak references and we can get away
with not allowing weak references at all.

[arc-inner]: https://github.com/rust-lang/rust/blob/1685c9298685f73db4fe890c1ed27b22408aaad7/src/liballoc/arc.rs#L249

`Shared` looks like this:

```rust
struct Shared {
    vec: Vec<u8>,
    original_capacity: usize,
    ref_count: AtomicUsize,
}
```

`AtomicUsize` is another atomic type - just like `AtomicPtr<T>` it can be safely
read and written from multiple threads. `vec` is the actual buffer, it's got a
second layer of indirection (since `Vec` is a pointer) so that it can be grown
in size without all the references to the `Shared` being invalidated. It would
be possible to store this data inline, but this would mean an unavoidable clone
when creating the data from a `Vec`. Maybe a future version of the library could
use a type parameter to allow the user to make this choice themselves, like the
[`SmallVec` crate does for its backing buffer][smallvec].

[smallvec]: https://docs.rs/smallvec/0.3.3/smallvec/struct.SmallVec.html

`original_capacity` is a bit of an odd field, it stores the capacity of the
vector that this was created from.
