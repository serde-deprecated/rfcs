- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Serde Issue: (leave this empty)

# Summary

One paragraph explanation of the feature.

# Motivation

**TLDR**: Let's do a clean restart of number serialization fitting everyone's needs. Skip forward to *Proposal*

There is the set `A` of number representations that the serialization formats use. Then there's the set `B` of number representations in rust. `B` also contains arbitrary user types (`BigNum`, `d128`, ...).

Ideally we'd have a function `f(A) -> B` and `f'(B) -> A`. Obviously that makes little sense, as it would require serde to know about all possible types. So instead we have a set `S` that contains all basic rust number types and  `f(A) -> S`, `f'(S)->A` as well as `g(B) -> S`, `g'(S) -> B`.

Right now we are discussing what `B` should contain and how to represent it.
The problem I'm seeing is that we are discussing the memory representation instead of the abstract representation.
There will never be a perfect memory representation that fits all needs (e.g. `BigNum` can represent all numbers, but requires memory allocation).
We'll always have to do tradeoffs.
The tradeoff decision shouldn't really be made by `serde` or an implementor, but by the final user.
We could of course get there through a huge `cfg!` madness, but a clean redesign should satisfy everyone's needs without overhead.

# Detailed design

We add a new trait `NumericSerialization` that is implemented for the regular rustc types. Users can implement this trait for their own types. Also there's a type `NumericTypeInfo` that contains various things that are useful to know about the type.

```rust
struct NumericTypeInfo<T: NumericSerialization> {
    name: &'static str,
    // if you have to hold more than 2^64 bits, then I have bad news for you.
    // None for infinitely sized types
    min_bits: Option<u64>,
    min_value: Option<&'static T>,
    // can a value of this type be signed
    signed: bool,
    // can there only be integral values? (only zeros after the comma)
    integral: bool,
}

/// `Display` bound for textual representations
trait NumericSerialization: core::fmt::Display {
    /// yields a value of the above struct
    /// FIXME: use associated constant once that's stable
    fn type_info() -> &'static NumericTypeInfo<Self>;
    /// yields the number of bits required to hold this *value*
    fn num_bits(&self) -> u64;
    /// whether the *value* is smaller than `0`
    fn is_signed(&self) -> bool;
    /// whether the *value* is integral
    fn is_integral(&self) -> bool;
    /// yield an f64 representation of the value if that can be done losslessly
    fn as_f64(&self) -> Option<f64>;
    /// yield an u64 representation of the value if that can be done losslessly
    fn as_u64(&self) -> Option<u64>;
    /// yield an i64 representation of the value if that can be done losslessly
    fn as_i64(&self) -> Option<i64>;
    /// yields the integral and decimal parts of this value
    /// FIXME: use `impl NumericSerialization` instead of `Box<NumericSerialization>`
    fn split_at_decimal(&self) -> (Box<NumericSerialization>, Box<NumericSerialization>);
    /// Iterate over the bits starting with the least significant one
    /// This function uses a closure until we get `impl Trait` return types
    /// FIXME: add more bounds to the `Iterator` to allow things like `.rev()`
    fn iter_bits<F: FnOnce(&mut Iterator<Item=bool>)>(&self, f: F);
}
```

With this information any numeric type can offer enough information for a `Serializer` to decide how to proceed.
The `Serializer` would then remove all `serialize_*` methods for integers and floats and instead have a single
`fn serialize_numeric<V>(&mut self, value: &V) where V: NumericSerialization` that one calls with the value that
is to be serialized.
It's important to use static dispatch when passing this value (there is no other way anyway, because it's not object safe, let's keep it this way).
Static dispatch helps optimizing out the indirections.

Deserialization needs the design in reverse. The trait `NumericDeserialization` is again implemented for concrete types.

```rust
trait NumericDeserialization {
    /// FIXME: use associated constant once that's stable
    fn type_info() -> &'static NumericTypeInfo<Self>;
    fn from_str(&str) -> Self;
    fn from_bits(&mut Iterator<Item=bool>) -> Self;
    fn from_u64(u64) -> Self;
    fn from_f64(f64) -> Self;
    fn from_i64(i64) -> Self;
    fn from_decimal(int: &NumericDeserialization, dec: &NumericDeserialization) -> Self;
}
```

The `Deserializer` trait would then have a method `fn deserialize_numeric<V>(&mut self, visitor: V) where V::Value: NumericDeserialization`.

# Drawbacks

More layers of indirection. Newcomers to serde that want to manually build `Serializer` and `Serialize` impls will have much more initial Code to read through.

# Alternatives

## Bignum

Can represent any number, but requires heap allocations

## String rep

Can represent any number, but some formats (like bincode) have a binary two complement representation, converting that to a string is not a good solution.

# Unresolved questions

* [ ] address serialization
* [ ] create prototype impl to verify that the Design is possible at all
