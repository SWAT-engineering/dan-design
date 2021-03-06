# DAN Proposal: Struct Style

## Types

This is DAN's type lattice:

						thing
				             	  |
		         -----------------------------------------
		         |            |        |       |         |
		       token         bool     int    string.   array
                       |
        ---------------------------------------------
        |               |        |     |      |     |
     user-define       u4      u8    u16    u32   u64
    

User-defined types are those defined via the `struct` or `choice` keyword.

Notice there are token types and simple types such as `bool, int, string`. Only descendants of `token` act also as parsers. This distinction is important, as allows us to create useful data structures that do not dependent on parsing.

# Execution model

A program consists of a series of types definitions, that in some cases, as said before, can be interpreted as parsers.  A program starts being executed by parsing a token/type definition such as:

```
def OneByte: u8
```

This program tries to parse a byte. The type to parse can be also composed type, such as a `struct`:

```
def Name:  struct{
	firstName: u8[20]
	secondName: u8[20]
}

def Info: struct{
	name: Name
	email: u8[30]
}
```

`Name` and `Info` defines types. If we start the execution at `Info`, the result of that execution will be a data structure containing fields name and email.


# Primitive data types


A simple program:

```
def OneByte: u8
```

It defines a type `OneByte` that works as an alias of `u8`. It also defines a parser that will exactly parse 1 byte. Therefore, we can expect `u8` and `OneByte` as interchangeable.

However, remember that DAN definitions are two-pronged: they define types and  parsers. Consider:

```
def TwoBytes: u8[2]
```

It defines a type `OneByte` that works as an alias of `u8[2]`. It also defines a parser that will exactly parse 2 bytes. However, from a typing perspective, `TwoBytes` is only interchangeably with `u8[]`. There is no dependent typing mechanism to track array sizes. This can be more complex if we add coniditions:

```
def A: u8 ?(this == 65)
```

It defines types `A` that works as an alias of `u8 ?(this == 65)`. It also defines a parser that will exactly parse 1 byte that must be the `A` character . However, from a typing perspective, `A` is only interchangeably with `u8`. There is no dependent typing mechanism to track conditions. Notice that the special variable `this` in the condition refers to the type/parser currently being defined.

# Struct types

We can also define structured data:

```
def Simple: struct{
	first: u8
	second: u16
}
```

Type `Simple` defines a product type composed of fields first and second. Let us define another type :

```
def Simple2: struct{
	first: u8
	second: u16
}
```
Types `Simple` and `Simple2` are not interchangeable. In other words, when defining structs, there is nominal and no structural typing. Besides that, the double notion of type/parsing is more evident in the case of structs, in the sense that the fields' order matters. When used as a parser, both `Simple` and `Simple2` will first parse a byte that will be assigned to field `first`, and then two consecutive bytes that will be assigned to `second`.

We cannot define new non-token types, but instead we can use them to create non-mutable variables (they therefore need to be always initialized). For instance:

```
val one: int = 1
```

A non-token type can be mixed in a struct with token types:

```
def Mixed: struct{
	token: u8
	myOne: int = 1
}
```

A non-token type can be used to compute derived attributes in a user-defined token type:

```
def DerivedExamle: struct{
	token1: u8[]
	size1: int = token1.length
	token2: u8[]
	tokenOffset: int = token2.offset
}
```

**Important:**  all array types have an implicit `length` field, and all token types, an implicit `offset` field.

A derived attribute can also be of a token type, working as an alias. The key distinctive syntactic feature in these cases is, again, the use of `=`:

```
def Inner: struct{
	n: u8
}

def DerivedExamle: struct{
	inner: Inner
	
	number: u8 = inner.n
}
```


For readability purposes, it is sometimes preferable to separate the token fields from the derived ones, as the token ones ``define'' the parser. To do so, a dependency graph is calculated and therefore programmers can arrange the derived fields in any order. Coming back to the previous example:


```
def DerivedExample2: struct{
	token1: u8[]
	token2: u8[]
	
	size1: int = token1.length
	tokenOffset: int = token2.offset
}

def DerivedExample3: struct{
	size1: int = token1.length
	tokenOffset: int = token2.offset
	
	token1: u8[]
	token2: u8[]
}
```


We can modularize programs by having parametric types:

```
def MustBeOfCertainAge(age: Boolean): struct{
	name: u8[20]
	age: u8 ? ( this == age)
}
```

For example, a token of type `MustBeOfCertainAge(1)` is interchangeable with any other that has the same parametric type. (**Note:** Is this consistent with our description of types above?).s

Sometimes, a field can be needed just for the parsing but it does not really need to have a named assigned. In that case, we can use the field name `_` or simply omit the left side hand of the field completely:

```
def Ignoring: struct{
	   name: u8[20]
	   _: u8 ? (this == ' ') // there must be a space after the name, but we do not name it
	   age: u8[2]
	   u8 ? (this == ' ') // there must be a space after the age, but we do not name it
   }
```


# Choice types

We can also define choice types:

```
def B: u8 ?(this == 66)

def AorB: choice{
	a: A
	b: B
}
```

Choice types imply backtracking. In this case, first an `A` will be tried to be parsed, and if not a `B`, in that declaration order. In other words, by default, the disambiguation is driven by the parsing process. We can also create an choice that is not dependent on the parsing. In that case, we have to put conditions on each member of the choice. For example:

```
def SimpleChoice(x: int): choice{
	[x == 0] zero: u8 ?(this == 60)
	default other: u8 ?(this >= 61 && this <= 71)
}
```

In this case, the parsing will be directed by a condition. If the parameter is 0, then the first alternative will be tried; otherwise, the second one, in that order. Notice that no automatic backtracking takes place in this case, since the parsing is directed by the conditions enclosed in brackets.

Choice types might also have derived fields. In that case, we need to declare them in the outer scope, and assign them in each variant:

```
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
```

# Meta properties

Properties that have to do with how to parse the token types are represented as annotations at the declaration level:

```
def Simple1: struct@(encoding=LittleIndian){
	content: u8[]
}
```

This means that Little Indian will be the encoding used for parsing content to produce a value of type `Simple1`.

Another meta-property is `offset`, that creates a new parsing pointer (disconnected from the context) and moves that parsing pointer to the specific (absolute) offset. If omitted, the default is to advance the normal pointer of the parent. Here an example where we start parsing from the 10th byte on:

```
def Root : struct {
    // parsing pointer is at position 0
    u8 oneByte
    // parsing pointers is at position 1
    u32 oneInt
    // parsing pointers is at position 5
    Simple2 sim
    // parsing pointers is at position 5
    Simple2 sim
    // parsing pointers is at position 5
    u8 oneByte
    // parsing pointers is at position 6
}

def Simple2: struct@(offset=9){
    // parsing pointers is at position 9
    content: u8[4]
    // parsing pointers is at position 13
}
```

# Type parameters

In certain ocassions, we need to write code that is type-generic, for example:

```
def GenericStruct<T> = struct{
	tentimes: T[10]
}
```

We can bind the type parameter obtaining different parsers for 10-element data:

```
def Name = u8[20]

def TenOfEverything = struct{
	tenNumbers: GenericStruct<u8>
	tenNames: GenericStruct<Name>
}
```

# Reified types

We can obtain the type of a value by using `.type`. An example of the utility of this is nested parsing:

```
def parse<T>(typ:type<T>, content:u8[]): T
```

`parse` is a built-in function that receives a reified type `type<T>` and an array of bytes and produces an element of type `T` by parsing the bytes according to the parser that type `T` defines.

For example:

```
def DescriptionAndContent<T>(parser: type<T>): struct{
	def description: u8[10]
	def content: u8[100]
	def parsedContent: T = parse<T>(parser, content)
}
```

In this case, the derived field `parsedContent` holds a reference to a value of type `T`, produced by parsing the content of field `content` using the parser defined by reified type `T`. This is an example of a client for `HeaderAndContent`.

```
def PNG: struct{
	first: u8[3] ? (this == "PNG")
	header: u8[25]
	content: u8[]
}

def Image: DescriptionAndContent<PNG>(PNG.type)
```

**TODO:** Decide whether we use type parameters or not. If not, we can represent the examples of this section as follows:

```
def parse(typ:type, content:u8[]): Token

def DescriptionAndContent(parser: type): struct{
	def description: u8[10]
	def content: u8[100]
	def parsedContent: Token = parse(parser, content)
}

def PNG: struct{
	first: u8[3] ? (this == "PNG")
	header: u8[25]
	content: u8[]
}

def Image: DescriptionAndContent(PNG.type)
```

The main disadvantage of this encoding is that we cannot enforce that `DescriptionAndContent.parsedContent` is of a specific type, and instead, we can just assume that it is the top token type `Token`. Is this really a problem?


# Type cohercions

How byte token types such as `u8` interact with simple types such as `int`?


# Metal constructs mapped

## Tokens
| Metal shorthand | Description | DSL |
| --- | --- | --- |
| `def(name, size)` | name a field with a size in bytes | `name : u8[size]` or `name: u32` |
| `nod(size)` | a ignored set of bytes | `_ : u8[size]` or `u8[size]` |
| `cho(name, list[Token])` | find the first token that parses successfully | `def name = choice { .. }` or inline as `x : choice { ... }` |
| `rep(name, Token)` | name a field that repeats a token until it fails to parse | `name: T[]` |
| `repn(name, Token, size)` | name a field that repeats a token n times | `name: T[n]` |
| `seq(name, list[Token])` | define a sequence of tokens | `def name = struct { .. }` |
| `sub(name, startAt, token)` | parse a token at a specific offset | `struct@(offset=startAt) { .. }` (not that this feature splits up the naming and the defining of the offset) |
| `pre(name, Token, predicate)` | unsure what it does | |
| `post(name, Token, predicate)` | unsure what it does | |
| `whl(name, Token, predicate)` | unsure what it does | |
| `opt(name, Token)` | unsure what it does | |
| `token(name)` | get a reference to a token "type" instead of a value | `T.type` (or `T.token`) |
| `tie(name, Token, data)` | Run a token parser on the result of a data expression | `name: parse(Token, data)`|
| `until(name, initialSize?, stepSize?, maxSize?, terminatorToken)` | unclear, the size params are all optional | |
| `when(name, Token, predicate)` | unsure | |

## Expression

| Metal shorthand | Description | DSL |
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
