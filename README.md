remodel library
===============

remodel is a lightweight C++ library that allows creating wrappers for
proprietary data structures and classes (with possibly many unknown fields)
of closed source applications or network traffic avoiding padding fields or
messy casts.

### Core aspects
- Easy to use
- Modern (C++14, limited by what is already supported by MSVC 12)
- Lightweight
  - Header-only library
  - No RTTI required
  - Exceptionless
- CMake, cross-platform support, tested on:
  - MSVC 12 aka. 2013, 14 aka. 2015 (Windows)
  - clang 3.6, 3.7 (MSVC emulation mode on Windows, OS X)

### Example
Imagine a scenario where you have instances of `Dog` in memory (let's say in
your dog-simulator game that you intend to write mods for) that need be
accessed.

```c++
class CustomString
{
  char *data;
  std::size_t length;
public:
  const char* c_str() { return data; }
  std::size_t size() { return length; }
};

class Dog
{
  CustomString name;
  CustomString* race;
  // ..
  // possibly many other unknown fields here
  // ..
  uint8_t age;
  bool hates_kittehz;
public:
  virtual int calculate_fluffiness() { /* ... */ }
  virtual void give_goodie(int amount) { /* ... */ }
  // .. more methods ..
};
```

Now the remodeled version:
```c++
class CustomString : public AdvancedClassWrapper<8 /* struct size */>
{
  REMODEL_ADV_WRAPPER(CustomString)
  // Note we omit the privates here because we decided we only need the methods.
public:
  MemberFunction<const char*(*)()> c_str{this, 0x12345678 /* function addr */};
  MemberFunction<std::size_t(*)()> size{this, 0x87654321};
};

// We don't create array or pointer fields referring to `Dog`, so we don't
// have to know it's size and can simply use `ClassWrapper` rather than
// `AdvancedClassWrapper`.
class Dog : public ClassWrapper
{
  REMODEL_WRAPPER(Dog)
  // We cheat and make the private fields public for our mod.
public:
  Field<CustomString> name{this, 0 /* struct offset */};
  Field<CustomString*> race{this, 8};
  // Note that we can just omit the unknown fields here without breaking
  // the integrity of the struct. No padding required.
  Field<uint8_t> age{this, 124};
  Field<bool> hates_kittehz{this, 125};
public:
  VirtualFunction<int(*)()> calculate_fluffiness{this, 0 /* vftable index */};
  VirtualFunction<void(*)(int)> give_goodie{this, 4};
};
```

And that's it! You can now use these wrappers pretty similar to how you would
use the original class.
```c++
auto dog = wrapper_cast<Dog>(dog_instance_location);
// Don't give the bad dog too much of the good stuff!
dog.give_goodie(dog.hates_kittehz ? 2 : 7);
// Year is over, +1 to age.
++dog.age;
// What was it's race again?
auto race = dog.race->toStrong().c_str();
```

Note that this library is in an alpha stage, so some things might not yet work
exactly like in the example and can change in the future.
