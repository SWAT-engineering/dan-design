# DAN Proposal

DAN is a domain-specific language for parsing binary data.

## Types

This is DAN's type lattice:

						

	    	         token         bool     int    string   array
                       |
        ---------------------------------------------
        |               |        |     |      |     |
     user-defined       u4      u8    u16    u32   u64
    

User-defined types are those defined via the `struct` or `choice` keyword.

Notice there are token types and simple types such as `bool, int, string`. Only descendants of `token` act also as parsers. This distinction is important, as allows us to create useful data structures that do not dependent on parsing.

# Execution model

A program consists of one or more modules. A module, thus, starts with a declaration of its name, followed by the modules it depends on:

```
module foo

import bar1
import bar2

...
```

The body of the module consists of a series of type definitions, that can be interpreted as parsers.  A program starts being executed by parsing a stream of bytes using a token/type definition, such as:

```
struct Name{
	u8[] firstName[20]
	u8[] secondName[20]
}
```
`Name` defines a composed type and also specifies a parser. If we start the execution at `Name`, the result of that execution/parsing will be a data structure containing fields `firstName` and `secondName`. Consider another example of a type definition:

```
struct Info{
	Name name
	u8[] email[30]
}
```

`Info` defines another type, that depends on `Name`. If we start the execution at `Info`, the result of that execution will be a data structure containing fields `name` of the user-defined type `Name`, and field `email` of the primitive type `u8[]`.

# Defining and using types in DAN

In DAN, types can be either user-defined or primitive. In this section, we review the available primitive types and how to define new user-defined types.

## Primitive token types


<!--A simple program:

```
def OneByte: u8
```

It defines a type `OneByte` that works as an alias of `u8`. It also defines a parser that will exactly parse 1 byte. Therefore, we can expect `u8` and `OneByte` as interchangeable.

Most DAN definitions are two-pronged: they define types and  parsers. Consider:

```
u8[2] TwoBytes
```

-->

Consider the following field definitions:

```
u8 oneUnsignedByte
s8 oneSignedByte
```

The first component of each definition is the type of the field, in these cases `u8` for an unsigned byte and `s8` for a signed byte. These definition can also be interpreted as parsers that will exactly parse 1 byte.

Now consider:

```
u16 twoUnignedBytes
u16 twoSignedBytes
```

They represent two bytes. The `u` and `s` types can be generalized to any multiple of 8, e.g. `u256` `s64`. An alternative way of defining sequential data is using lists:

```
u8[] twoBytes[2]
```

The first component of this definition is the type of the field, in this case, a list of unsigned bytes, followed by the name of the field and then, optionally, the initial size of the list. In this case, `twoBytes` will exactly parse 2 bytes. Alternatively, we can write:

```
u8[] aList
```

Field `aList` will contain a sequence of bytes of arbitrary length.

Field definitions can also include conditions 
```
u8 a ?(this != 65)
```

It defines field `a` of type `u8`. The value will only be assigned if at execution time, the parsed byte is not equal to 65, or, from another perspective, the parsed character is different from `A` (the character corresponding to 65). Notice that the special variable `this` in the condition refers to the field currently being defined.

## Struct types

We know than we can define structured data that enrich the set of available types with user-defined ones:

```
struct Simple{
	u8 first
	u16 second
}
```

This definition introduces the user-defined type `Simple`, that corresponds to a composed type that contains fields first and second. The double notion of type/parsing is more evident in the case of structs, in the sense that the fields' order matters. When used as a specification for parsing, `Simple` will first parse a byte that will be assigned to field `first`, and then two consecutive bytes that will be assigned to `second`.


## Primitive non-token types

Not every type is parseable. Non-token types allow us to represent common primitive values that are not attached to a particular byte representation, such as strings, integers or booleans. As they do not define parsers, fields of a non-token type need to be initialized at the moment of declaration, for example:

```
struct Simple{
	int myOne = 1
}
```

A non-token type can be used to compute derived attributes inside an user-defined token type:

```
struct DerivedExample{
	u8[] token1[3]
	int offset1 = token1.offset
	u8[] token2
	int length2 =  token2.length
}
```

**Important:**  all array types have an implicit `length` field, and all token types, an implicit `offset` field.

<!--A derived attribute can also be of a token type, working as an alias. The key distinctive syntactic feature in these cases is, again, the use of `=`:

```
def Inner: struct{
	n: u8
}

def DerivedExamle: struct{
	inner: Inner
	
	number: u8 = inner.n
}
```
-->

For readability purposes, it is sometimes preferable to separate the token fields from the derived ones, as the token ones ``define'' the parser. To do so, a dependency graph is calculated and therefore programmers can arrange the derived fields in any order. Coming back to the previous example, the following struct definitions are two valid alternative encodings:


```
struct DerivedExample2{
	u8[3] token1
	u8[] token2
	
	int offset1 = token1.offset
	int length2 = token2.length
}

struct DerivedExample3{
	int offset1 = token1.offset
	int length2 = token2.length
		
	u8[3] token1
	u8[] token2
}
```


## Choice types

We can also define choice types:

```
struct A{
	u8 content ? (u8 == "A")
}

struct B{
	u8 content ? (u8 == "B")
}

choice AB{
	A a
	B b
}
```

Choice types imply backtracking. In this case, an attempt to parse the input with type `A` will be made first. If this parsing does not succeed, then `B` will be tried, in strict declaration order. In other words, the disambiguation is driven by the parsing process. Note that the fields in a choice correspond to alternatives, therefore, they are not named.

Notice too that despite both alternatives featuring a field `content` of type `u8`, this field is not accessible if we have a reference to a value of type `AB`. In order to do this, we need to use _abstract fields_. An abstract field is a field declared inside a choice, whose implementation is abstract and must be defined in *each one of the alternatives*. Let us redefine `AB` using this concept:

```
choice AB2{
	abstract u8 content
	A a
	B b
}
```

Since both alternatives have a field called content of type `u8`, this definition is valid. Thus, `AB2` exposes a field content that will hold the proper value, depending on which alternative is going to be taken.

<!--```
Choice types might also have derived fields. In that case, we need to declare them in the outer scope, and assign them in each variant:


def DerivedChoice: choice{
	size: int
	variant1: struct{
		a: u8? (this == 65)
		rest: u8[2]
		size = 3
	}
	variant2: struct{
		a: u8?
		rest: u8[3]
		size = 4
	}
}
```-->

## Constructors of user-defined types

We can modularize programs by having parametric user-defined types:

```
struct MustBeOfCertainAge(int ageLimit){
	u8[20] name
	u8 age ? (this == ageLimit)
}
```

In this case, `MustBeOfCertainAge` acts as a constructor receiving one integer argument. 

In order to build parsers conforming to this definition we need to parameterize the construction of values of this type. Consider, for instance, this field declaration:

```
MustBeOfCertainAge person(18)
```

This field definition instantiate the `MustBeOfCertainAge` struct definition for the concrete argument `18`.

## Implicit coercions

In the previous example, notice the equality check performed in the field `age` declaration within the `MustBeOfCertainAge` definition. We are comparing `this` (which refers to the field `age` of type `u8` with `ageLimit`, defined as an `int` parameter. This comparison works because there is an implicit cohercion from the type `u8` to the type `int`. Primitive token types, such as `u` types, `s` types, or list types containing them, can be coerced to primitive non-token type, such as `string` or `int` by using some implicit encoding conventions. To alter these coventions, we can use "meta properties", explained below.
 
## Anonymous fields

Sometimes, a field is needed just for parsing purposes but it does not really need to have a named assigned. In that case, we can use the field name `_`, which means to ignore that field when accessing the fields of a value of such type:

```
struct Ignoring{
	   u8[20] name
	   u8 _ ? (this == ' ') // there must be a space after the name, but we do not name it
	   u16 age
	   u8 _ ? (this == ' ') // there must be a space after the age, but we do not name it
   }
```

## Inline user-defined types

In the choice example, for defining types `AB` and `AB2` we also needed to define types `A` and `B`, exclusively in order to define the alternatives of the choice. In these situations, it might be more convenient to inline the struct definitions, as shown here:


```
choice AB3{
	abstract u8 content
	struct{
		u8 content ? (u8 == "A")
	}
	struct{
		u8 content ? (u8 == "B")
	}
}
```

The same applies for a struct definition, for example:

```
struct Info2{
	struct{
		u8[] firstName[20]
		u8[] secondName[20]
	} name
	u8[] email[30]
}
```

In order to access the `firstName` field of the inner struct, assuming that we have a variable `i` of type `Info2`, we need to write `anInfo2.name.firstName`.


# Meta properties

Properties that have to do with how to parse the token types are represented as annotations at the declaration level:

```
struct Block@(encoding=LittleIndian){
	u8[] content
}
```

This means that Little Indian will be the encoding used for parsing content to produce a value of type `Block`.

Another meta-property is `offset`, that creates a new parsing pointer (disconnected from the context) and moves that parsing pointer to the specific (absolute) offset. If omitted, the default is to advance the normal pointer of the parent. Here an example where we start parsing from the 10th byte on:

```
struct Root {
    // parsing pointer is at position 0
    u8 oneByte
    // parsing pointers is at position 1
    u32 oneInt
    // parsing pointers is at position 5
    Block2 sim
    // parsing pointers is at position 5
    Block2 sim
    // parsing pointers is at position 5
    u8 oneByte
    // parsing pointers is at position 6
}

struct Block2@(offset=9){
    // parsing pointers is at position 9
    u8[4] content
    // parsing pointers is at position 13
}
```

# Metal constructs mapped to DAN

## Tokens
| Metal shorthand | Description | DAN |
| --- | --- | --- |
| `def(name, size)` | name a field with a size in bytes | `u8[size] name` or `u32 name` |
| `nod(size)` | a ignored set of bytes | `u8[size] _`|
| `cho(name, list[Token])` | find the first token that parses successfully | `choice name{ ... }`|
| `rep(name, Token)` | name a field that repeats a token until it fails to parse | `T[] name` |
| `repn(name, Token, size)` | name a field that repeats a token n times | `T[] name[n]` |
| `seq(name, list[Token])` | define a sequence of tokens | `struct name{ ... }` |
| `sub(name, startAt, token)` | parse a token at a specific offset | `struct@(offset=startAt) { ... }` (note that this feature splits up the naming and the defining of the offset) |
| `pre(name, Token, predicate)` | to be determined | |
| `post(name, Token, predicate)` | to be determined | |
| `whl(name, Token, predicate)` | to be determined | |
| `opt(name, Token)` | to be determined | |
| `token(name)` | get a reference to a token "type" instead of a value | `T.type` |
| `tie(name, Token, data)` | Run a token parser on the result of a data expression | `u8[] name = parse(token, data)` (`parse` is a native function of signature `u8[] parse(typ tokenType, u8[] data)`)|
| `until(name, initialSize?, stepSize?, maxSize?, terminatorToken)` | unclear, the size params are all optional | |
| `when(name, Token, predicate)` | unsure | |

## Expression

| Metal shorthand | Description | DAN |
| --- | --- | --- |
| `add` | | `l + r` |
| `div` | | `l / r` |
| `mul` | | `l * r` |
| `sub` | | `l - r` |
| `mod` | | `l % r` |
| `neg` | | `-l` |
| `shl` | | `l << r` |
| `shr` | | `l >> r` |
| `and` | binary and | `l & r` |
| `or` |  binary or | `l & r` |
| `not` |  binary (and boolean) not | `!l` |
| `and` | boolean and | `l && r` |
| `or` |  boolean or | `l && r` |
| `eq* | comparision operators | default comparison operators |
| `con` | constants | literal syntax for arbitrary precision ints, strings, arrays |
| `len(l)` |  | `l.length` |
| `ref(name)` | create a list of all instances of something with that name | `x.name` (however, it's not dynamic scoping anymore) |
| `first(l)` | first element of the list/array | `l[0]` |
| `last(l)` | last element of the list/array | `l[-1]` or `l[l.length - 1]` |
| `nth(l, i)` | nth element of the list/array | `l[i]` |
| `offset(reference name)` | get the absolute offset of something that has already been parsed | `n.offset` (without dynamic scoping aspect) |
| `cat(a, b)` | concat the bytes together of two parsed trees | `a + b` |
| `elvis(v, orElse)` | a bit unclear what it exactly does, as it's mixed with the everything is a list concept of metal | |
| `count(t)` | the size of  a list | unclear, as we've removed the concepts of lists |
| `foldLeft(values, reducer, initial?)` | unclear if still needed after removing lists concept | |
| `foldRight(values, reducer, initial?)` | unclear if still needed after removing lists concept | |
| `fold(values, reducer, initial?)` | unclear if still needed after removing lists concept | |
| `mapLeft(values, mapper, left, rightExpand)` | unclear if still needed after removing lists concept | |
| `mapRight(values, mapper, leftExpand, right)` | unclear if still needed after removing lists concept | |
| `rev(values)` | reverse a list | unclear if still needed |
| `exp(base, count)` | repeat (expand) `base` `count` times | missing, maybe something with slices? |
| `bytes(x)` | get the bytes represented by something | automatic cohercion solves this |