- Start Date: 2015-11-18
- RFC PR: (leave this empty)
- Serde Issue: (leave this empty)

# Summary

Add hooks hooks to enable pointer cycles to be efficiently serialized, and
allow those cycles to be rebuilt upon deserialization.

# Motivation

Serde right now supports serializing pointer cycles (by way of `Rc` and `Arc`
smart pointers), but it does so naively by just duplicating the underlying
pointer. This not only causes the serialized format to potentially use more
space than necessary, but also forgets the cyclic nature of these values upon
deserialization. Even worse, if a pointer cycle forms a loop, then Serde will
fall into an infinite loop that will either result in it consuming all the
memory or disk space on the machine as it keeps serializing the same structure
over and over.

To prevent this, Serde serializers and deserializers should be extended to
expose the potentially cyclic nature of these pointers.

# Detailed design

This RFC proposes adding the following methods:

```rust
pub trait Serializer {
    ...

    #[inline]
    fn serialize_cyclic_ref<T>(&mut self, v: &T) -> Result<(), Self::Error>
        where T: Serialize,
    {
        v.serialize(self)
    }

    ...
}

impl<T> Serialize for Rc<T> where T: Serialize, {
    #[inline]
    fn serialize<S>(&self, serializer: &mut S) -> Result<(), S::Error>
        where S: Serializer,
    {
        serializer.serialize_cyclic_ref(**self)
    }
}

pub trait Deserializer {
    ...

    #[inline]
    fn deserialize_cyclic_ref<V>(&mut self, visitor: V) -> Result<V::Value, Self::Error>
        where V: Visitor,
    {
        self.deserialize(visitor)
    }
}


impl<T: Deserialize> Deserialize for Rc<T> {
    fn deserialize<D>(deserializer: &mut D) -> Result<Rc<T>, D::Error>
        where D: Deserializer,
    {
        let val = try!(Deserialize::deserialize_cyclic_ref(deserializer));
        Ok(Rc::new(val))
    }
}
```

This should be enough to implement a format that can handle cycles. This can be
done by serializing into an intermediate structure and a side table to hold the
cyclic pointers. For example:

```rust
extern crate serde_json;

use std::collections::hash_map::{self, HashMap};

struct SerializerWithCycles {
    ser: serde_json::value::Serializer,
    cycle_map: HashMap<usize, usize>,
    cycles: Vec<serde_json::Value>,
}

impl Serializer for SerializerWithCycles {
    // proxy other methods to `self.ser`...

    fn serialize_cyclic_ref<T>(&mut self, v: &T) -> Result<(), Self::Error>
        where T: Serialize,
    {
        match self.cycle_maps.entry(v as usize) {
            hash_map::Entry::Occupied(entry) => {
                self.ser.serialize(entry.get())
            }
            hash_map::Entry::Vacant(entry) => {
                let id = self.cycles.len();
                let v = try!(serde_json::to_value(v));
                self.cycles.push(v);
                entry.insert(id);
                Ok(())
            }
        }
    }
}

pub fn to_writer<W, T>(wr: &W, v: &T) -> serde_json::Result<()>
    where W: Writer,
          T: Serialize,
{
    let serializer = SerializerWithCycles { ... };
    try!(serializer.serialize(v));
    let SerializerWithCycles { ser, cycles, _ } = serializer;

    let json = ser.unwrap();

    serde_json::to_writer(wr, (json, cycles))
}
```

Deserialization is a little more complicated, since it needs to maintain a
cache of `Any` objects to hold the cyclic pointers.

```rust
extern crate serde_json;

use std::any::Any;
use std::collections::hash_map::{self, HashMap};
use std::mem;

struct DeserializerWithCycles {
    de: serde_json::value::Deserializer,
    cycles: Vec<serde_json::Value>,
    cycle_map: HashMap<usize, Any>,
}

impl Deserializer for DeserializerWithCycles {
    // proxy other methods to `self.de`...

    fn deserialize_cyclic_ref<V>(&mut self, visitor: V) -> Result<V::Value, Self::Error>
        where V: Visitor,
              V::Value: Reflect + Clone,
    {
        let id = try!(usize::deserialize(self));
        let entry = match self.cycle_map.entry(id) {
            hash_map::Occupied(entry) => entry,
            hash_map::Empty(entry) => {
                let json_value_ref = self.cycles.get_mut(id) {
                    Some(value) => value,
                    None => { return Err(Error::syntax_error("missing reference")); }
                };

                let mut json_value = serde_json::Null;
                mem::swap(&mut json_value, json_value_ref);

                let value = try!(serde_json::from_value(json_value));
                entry.insert(value as Any)
            }
        };

        match entry.get().downcast_ref() {
            Some(value) => {
                Ok(value.clone())
            }
            None => {
                Err(Error::syntax_error("missing from typemap vec")
            }
        }
    }
}

pub fn from_wr<R, T>(rdr: R) -> Result<T, Error> {
    let (json, cycles) = try!(serde_json::from_reader(rdr));
    let mut de = DeserializerWithCycles {
        de: serde_json::value::Deserializer::new(json),
        cycles: cycles,
        cycle_map: HashMap::new(),
    };

    T::deserialize(&mut de)
}
```

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

* Is `*_cyclic_ref` a good name? How about `*_cycle`?
