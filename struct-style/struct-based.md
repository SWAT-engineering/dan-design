# DAN Proposal: Struct Style

## Types

This is DAN's type lattice:

							 	thing
				             	  |
		         -----------------------------------------
		         |            |        |       |         |
		       token         bool     int    string.   array
               |
      --------------------------------------
      |             |    |    |     |	   |
    user-defined   u4   u8   u16   u32 	  u64
    

User-defined types are those defined via the `struct` or `union` keyword.

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


# Examples


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

We can also define union types:

```
def B: u8 ?(this == 66)

def AorB: union{
	a: A
	b: B
}
```

Union types imply backtracking. In this case, first an `A` will be tried to be parsed, and if not a `B`, in that declaration order. In other words, by default, the disambiguation is driven by the parsing process. We can also create an union that is not dependent on the parsing. In that case, we have to put conditions on each member of the union. For example:

```
def SimpleUnion(x: int): union{
	[x == 0] zero: u8 ?(this == 60)
	default other: u8 ?(this >= 61 && this <= 71)
}
```

In this case, the parsing will be directed by a condition. If the parameter is 0, then the first alternative will be tried; otherwise, the second one, in that order. Notice that no automatic backtracking takes place in this case, since the parsing is directed by the conditions enclosed in brackets.

Union type might also have derived fields. In that case, we need to declare them in the outer scope, and assign them in each variant:

```
def DerivedUnion: union{
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


# Type cohercions

How byte token types such as `u8` interact with simple types such as `int`?


