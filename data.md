EVM Words
=========

EVM uses bounded 256 bit integer words.
Here we provide the arithmetic of these words, as well as some data-structures over them.

EVM also has bytes (8 bit words) for some encodings of data, but only a very light dependence on byte-level manipulation.
It's doubtful that having two different ways of reading the base data-units (or that limiting yourself to machine-representable units) is beneficial.
Arguably hardware should evolve to directly support more elegant languages, rather than our languages evolving towards our hardware.

```rvk
requires "krypto.k"
```

```k
requires "domains.k"

module EVM-DATA
    imports KRYPTO

    syntax KResult ::= Int
```

Primitives
----------

Primitives provide the basic conversion from K's sorts `Int` and `Bool` to EVM's sort `Word`.

-   `Int` is a subsort of `Word`.
-   `chop` interperets an interger module $2^256$.

```k
    syntax Word ::= Int | chop ( Word ) [function]
 // ----------------------------------------------
    rule chop( W:Int ) => chop ( W +Int (2 ^Int 256) ) requires W <Int 0
    rule chop( W:Int ) => chop ( W %Int (2 ^Int 256) ) requires W >=Int (2 ^Int 256)
    rule chop( W:Int ) => W requires W >=Int 0 andBool W <Int (2 ^Int 256)
```

-   `bool2Word` interperets a `Bool` as a `Word`.
-   `word2Bool` interperets a `Word` as a `Bool`.

```k
    syntax Word ::= bool2Word ( Bool ) [function]
 // ---------------------------------------------
    rule bool2Word(true)  => 1
    rule bool2Word(false) => 0

    syntax Bool ::= word2Bool ( Word ) [function]
 // ---------------------------------------------
    rule word2Bool( 0 ) => false
    rule word2Bool( W ) => true  requires W =/=K 0
```

-   `#ifWord_#then_#else_#fi` provides a conditional in `Word` expressions.

```k
    syntax Word ::= "#ifWord" Bool "#then" Word "#else" Word "#fi" [function]
 // -------------------------------------------------------------------------
    rule #ifWord true  #then W #else _ #fi => W
    rule #ifWord false #then _ #else W #fi => W
```

-   `sgn` gives the twos-complement interperetation of the sign of a `Word`.
-   `abs` gives the twos-complement interperetation of the magnitude of a `Word`.

```k
    syntax Int ::= sgn ( Word ) [function]
                 | abs ( Word ) [function]
 // --------------------------------------
    rule sgn(I:Int) => -1 requires I >=Int (2 ^Int 255)
    rule sgn(I:Int) => 1  requires I <Int (2 ^Int 255)

    rule abs(I:Int) => 0 -Word I requires sgn(I) ==K -1
    rule abs(I:Int) => I         requires sgn(I) ==K 1
```

-   `maxWord` and `minWord` are the lifting of `maxInt` and `minInt` (through the subsort monad!).

```k
    syntax Word ::= maxWord ( Word , Word ) [function]
                  | minWord ( Word , Word ) [function]
 // --------------------------------------------------
    rule maxWord(W0:Int, W1:Int) => maxInt(W0, W1)
    rule minWord(W0:Int, W1:Int) => minInt(W0, W1)
```

### Symbolic Words

-   `#symbolicWord` generates a fresh existentially-bound symbolic word.

Note: Comment out this block (remove the `k` tag) if using RV K.

```k
    syntax Word ::= "#symbolicWord" [function]
 // ------------------------------------------
    rule #symbolicWord => ?X:Int
```

Arithmetic
----------

-   `up/Int` performs integer division but rounds up instead of down.

```k
    syntax Int ::= Int "up/Int" Int [function]
 // ------------------------------------------
    rule I1 up/Int I2 => I1 /Int I2          requires I1 %Int I2 ==K 0
    rule I1 up/Int I2 => (I1 /Int I2) +Int 1 requires I1 %Int I2 =/=K 0
```

`logNInt` returns the log base N (floored) of an integer.

```k
    syntax Int ::= log2Int(Int) [function]
 // ------------------------------------------
    // log2Int is the index of the most significant bit of a number
    // log2Int(0) is undefined
    rule log2Int(1) => 0
    rule log2Int(W:Int) => 1 +Int log2Int(W >>Int 1)    requires W >Int 1

    syntax Int ::= log256Int(Int) [function]
 // ------------------------------------------
    rule log256Int(N:Int) => log2Int(N) /Int log2Int(256)
```

The corresponding `<op>Word` operations automatically perform the correct `Word` modulus.

```k
    syntax Word ::= Word "+Word" Word [function]
                  | Word "*Word" Word [function]
                  | Word "-Word" Word [function]
                  | Word "/Word" Word [function]
                  | Word "%Word" Word [function]
 // --------------------------------------------
    rule W0:Int +Word W1:Int => chop( W0 +Int W1 )
    rule W0:Int -Word W1:Int => chop( W0 -Int W1 )
    rule W0:Int *Word W1:Int => chop( W0 *Int W1 )
    rule W0:Int /Word 0      => 0
    rule W0:Int /Word W1:Int => chop( W0 /Int W1 ) requires W1 =/=K 0
    rule W0:Int %Word 0      => 0
    rule W0:Int %Word W1:Int => chop( W0 %Int W1 ) requires W1 =/=K 0
```

Care is needed for `^Word` to avoid big exponentiation.

```k
    syntax Word ::= Word "^Word" Word [function]
 // --------------------------------------------
    rule W0:Int ^Word W1:Int => (W0 ^Word (W1 /Int 2)) ^Word 2  requires W1 >=Int (2 ^Int 16) andBool W1 %Int 2 ==Int 0
    rule W0:Int ^Word W1:Int => (W0 ^Word (W1 -Int 1)) *Word W0 requires W1 >=Int (2 ^Int 16) andBool W1 %Int 2 ==Int 1
    rule W0:Int ^Word W1:Int => chop( W0 ^Int W1 )              requires W1 <Int (2 ^Int 16)
```

`/sWord` and `%sWord` give the signed interperetations of `/Word` and `%Word`.

```k
    syntax Word ::= Word "/sWord" Word [function]
                  | Word "%sWord" Word [function]
 // ---------------------------------------------
    rule W0:Int /sWord W1:Int => (abs(W0) /Word abs(W1)) *Word (sgn(W0) *Int sgn(W1))
    rule W0:Int %sWord W1:Int => (abs(W0) %Word abs(W1)) *Word sgn(W0)
```

Comparison Operators
--------------------

The `<op>Word` comparison operators automatically interperet the `Bool` as a `Word`.

```k
    syntax Word ::= Word "<Word"  Word [function]
                  | Word ">Word"  Word [function]
                  | Word "<=Word" Word [function]
                  | Word ">=Word" Word [function]
                  | Word "==Word" Word [function]
 // ---------------------------------------------
    rule W0:Int <Word  W1:Int => bool2Word(W0 <Int  W1)
    rule W0:Int >Word  W1:Int => bool2Word(W0 >Int  W1)
    rule W0:Int <=Word W1:Int => bool2Word(W0 <=Int W1)
    rule W0:Int >=Word W1:Int => bool2Word(W0 >=Int W1)
    rule W0:Int ==Word W1:Int => bool2Word(W0 ==Int W1)
```

-   `s<Word` implements a less-than for `Word` (with signed interperetation).

```k
    syntax Word ::= Word "s<Word" Word [function]
 // ---------------------------------------------
    rule W0:Int s<Word W1:Int => W0 <Word W1           requires sgn(W0) ==K 1  andBool sgn(W1) ==K 1
    rule W0:Int s<Word W1:Int => bool2Word(false)      requires sgn(W0) ==K 1  andBool sgn(W1) ==K -1
    rule W0:Int s<Word W1:Int => bool2Word(true)       requires sgn(W0) ==K -1 andBool sgn(W1) ==K 1
    rule W0:Int s<Word W1:Int => abs(W1) <Word abs(W0) requires sgn(W0) ==K -1 andBool sgn(W1) ==K -1
```

Bitwise operators.
------------------

Bitwise logical operators are lifted from the integer versions.

```k
    syntax Word ::= "~Word" Word        [function]
                  | Word "|Word" Word   [function]
                  | Word "&Word" Word   [function]
                  | Word "xorWord" Word [function]
 // ----------------------------------------------
    rule ~Word W:Int           => chop( W xorInt ((2 ^Int 256) -Int 1) )
    rule W0:Int |Word   W1:Int => chop( W0 |Int W1 )
    rule W0:Int &Word   W1:Int => chop( W0 &Int W1 )
    rule W0:Int xorWord W1:Int => chop( W0 xorInt W1 )
```

-   `bit` gets bit $N$ (0 being MSB).
-   `byte` gets byte $N$ (0 being the MSB).

```k
    syntax Word ::= bit  ( Word , Word ) [function]
                  | byte ( Word , Word ) [function]
 // -----------------------------------------------
    rule bit(N:Int, W:Int)  => (W >>Int (255 -Int N)) %Int 2
    rule byte(N:Int, W:Int) => (W >>Int (256 -Int (8 *Int (N +Int 1)))) %Int (2 ^Int 8)
```

-   `#nBits` shifts in $N$ ones from the right.
-   `#nBytes` shifts in $N$ bytes of ones from the right.
-   `_<<Byte_` shifts an integer 8 bits to the left.

```k
    syntax Int  ::= #nBits  ( Int )  [function]
                  | #nBytes ( Int )  [function]
                  | Int "<<Byte" Int [function]
 // -------------------------------------------
    rule #nBits(N)  => (2 ^Int N) -Int 1
    rule #nBytes(N) => #nBits(N *Int 8)
    rule N <<Byte M => N <<Int (8 *Int M)
```

-   `signextend(N, W)` sign-extends from byte $N$ of $W$ (0 being MSB).

```k
    syntax Word ::= signextend( Word , Word ) [function]
 // ----------------------------------------------------
    rule signextend(N:Int, W:Int) => chop( (#nBytes(31 -Int N) <<Byte (N +Int 1)) |Int W ) requires         word2Bool(bit(256 -Int (8 *Int (N +Int 1)), W))
    rule signextend(N:Int, W:Int) => chop( #nBytes(N +Int 1)                      &Int W ) requires notBool word2Bool(bit(256 -Int (8 *Int (N +Int 1)), W))
```

-   `keccak` serves as a wrapper around the `Keccak256` in `KRYPTO`.

```k
    syntax Word ::= keccak ( WordStack ) [function]
 // -----------------------------------------------
    rule keccak(WS) => #parseHexWord(Keccak256(#unparseByteStack(WS)))
```

Data Structures
===============

Several data-structures and operations over `Word` are useful to have around.

Word Stack
----------

EVM is a stack machine, and so needs a stack of words to operate on.
The stack and some standard operations over it are provided here.
This stack also serves as a cons-list, so we provide some standard cons-list manipulation tools.

```k
    syntax WordStack ::= ".WordStack" | Word ":" WordStack
 // ------------------------------------------------------
```

-   `_++_` acts as `WordStack` append.
-   `#take( N , WS)` keeps the first $N$ elements of a `WordStack` (passing with zeros as needed).
-   `#drop( N , WS)` removes the first $N$ elements of a `WordStack`.

```k
    syntax WordStack ::= WordStack "++" WordStack [function]
 // --------------------------------------------------------
    rule .WordStack ++ WS' => WS'
    rule (W : WS)   ++ WS' => W : (WS ++ WS')

    syntax WordStack ::= #take ( Word , WordStack ) [function]
 // ----------------------------------------------------------
    rule #take(0, WS)         => .WordStack
    rule #take(N, .WordStack) => 0 : #take(N -Word 1, .WordStack) requires word2Bool(N >Word 0)
    rule #take(N, (W : WS))   => W : #take(N -Word 1, WS)         requires word2Bool(N >Word 0)

    syntax WordStack ::= #drop ( Word , WordStack ) [function]
 // ----------------------------------------------------------
    rule #drop(0, WS)         => WS
    rule #drop(N, .WordStack) => .WordStack
    rule #drop(N, (W : WS))   => #drop(N -Word 1, WS) requires word2Bool(N >Word 0)
```

-   `WS [ N ]` accesses element $N$ of $WS$.
-   `WS [ N := W ]` sets element $N$ of $WS$ to $W$ (padding with zeros as needed).

```k
    syntax Word ::= WordStack "[" Word "]" [function]
 // -------------------------------------------------
    rule (W0 : WS)   [0] => W0
    rule (.WordStack)[N] => 0             requires word2Bool(N >Word 0)
    rule (W0 : WS)   [N] => WS[N -Word 1] requires word2Bool(N >Word 0)

    syntax WordStack ::= WordStack "[" Word ":=" Word "]" [function]
 // ----------------------------------------------------------------
    rule (W0 : WS)  [ 0 := W ] => W  : WS
    rule .WordStack [ N := W ] => 0  : (.WordStack [ N -Int 1 := W ]) requires word2Bool(N >Word 0)
    rule (W0 : WS)  [ N := W ] => W0 : (WS [ N -Int 1 := W ])         requires word2Bool(N >Word 0)

    syntax WordStack ::= WordStack "[" Word ".." Word "]" [function]
 // ----------------------------------------------------------------
    rule WS [ START .. WIDTH ] => #take(WIDTH, #drop(START, WS))
```

-   `#sizeWordStack` calculates the size of a `WordStack`.
-   `_in_` determines if a `Word` occurs in a `WordStack`.

```k
    syntax Int ::= #sizeWordStack ( WordStack ) [function]
 // ------------------------------------------------------
    rule #sizeWordStack ( .WordStack ) => 0
    rule #sizeWordStack ( W : WS )     => 1 +Int #sizeWordStack(WS)

    syntax Bool ::= Word "in" WordStack [function]
 // ----------------------------------------------
    rule W in .WordStack => false
    rule W in (W' : WS)  => (W ==K W') orElseBool (W in WS)
```

-   `#padToWidth(N, WS)` makes sure that a `WordStack` is the correct size.

```k
    syntax WordStack ::= #padToWidth ( Int , WordStack ) [function]
 // ---------------------------------------------------------------
    rule #padToWidth(N, WS) => WS                     requires notBool #sizeWordStack(WS) <Int N
    rule #padToWidth(N, WS) => #padToWidth(N, 0 : WS) requires #sizeWordStack(WS) <Int N
```

Byte Arrays
-----------

The local memory of execution is a byte-array (instead of a word-array).

-   `#asWord` will interperet a stack of bytes as a single word (with MSB first).
-   `#asByteStack` will split a single word up into a `WordStack` where each word is a byte wide.

```k
    syntax Word ::= #asWord ( WordStack ) [function]
 // ------------------------------------------------
    rule #asWord( .WordStack )    => 0
    rule #asWord( W : .WordStack) => W
    rule #asWord( W0 : W1 : WS )  => #asWord(((W0 *Word 256) +Word W1) : WS)

    syntax WordStack ::= #asByteStack ( Word )             [function]
                       | #asByteStack ( Word , WordStack ) [function, klabel(#asByteStackAux)]
 // ------------------------------------------------------------------------------------------
    rule #asByteStack( W ) => #asByteStack( W , .WordStack )
    rule #asByteStack( 0 , WS ) => WS
    rule #asByteStack( W , WS ) => #asByteStack( W /Int 256 , W %Int 256 : WS ) requires W =/=K 0
```

Addresses
---------

-   `#addr` turns an Ethereum word into the corresponding Ethereum address (160 LSB).

```k
    syntax Word ::= #addr ( Word ) [function]
 // -----------------------------------------
    rule #addr(W) => W %Word (2 ^Word 160)
```

Word Map
--------

Most of EVM data is held in finite maps.
We are using the polymorphic `Map` sort for these word maps.

-   `WM [ N := WS ]` assigns a contiguous chunk of $WM$ to $WS$ starting at position $W$.
-   `#asMapWordStack` converts a `WordStack` to a `Map`.
-   `#range(M, START, WIDTH)` reads off $WIDTH$ elements from $WM$ beginning at position $START$ (padding with zeros as needed).

```k
    syntax Map ::= Map "[" Word ":=" WordStack "]" [function]
 // ---------------------------------------------------------
    rule WM[ N := .WordStack ] => WM
    rule WM[ N := W : WS     ] => (WM[N <- W])[N +Word 1 := WS]

    syntax Map ::= #asMapWordStack ( WordStack ) [function]
 // -------------------------------------------------------
    rule #asMapWordStack(WS:WordStack) => .Map [ 0 := WS ]

    syntax WordStack ::= #range ( Map , Word , Word ) [function]
 // ------------------------------------------------------------
    rule #range(WM,         N, 0) => .WordStack
    rule #range(WM,         N:Int, M:Int) => 0 : #range(WM, N +Int 1, M -Int 1) requires (M >Int 0) andBool notBool N in keys(WM)
    rule #range(N |-> W WM, N:Int, M:Int) => W : #range(WM, N +Int 1, M -Int 1) requires (M >Int 0)
```

Parsing/Unparsing
=================

The EVM test-sets are represented in JSON format with hex-encoding of the data and programs.
Here we provide some standard parser/unparser functions for that format.

JSON
----

Writing a JSON parser in K takes 5 lines.

```k
    syntax JSONList ::= List{JSON,","}
    syntax JSON     ::= String
                      | String ":" JSON
                      | "{" JSONList "}"
                      | "[" JSONList "]"
 // ------------------------------------
```

Parsing
-------

These parsers can interperet hex-encoded strings as `Word`s, `WordStack`s, and `Map`s.

-   `#parseHexWord` interperets a string as a single hex-encoded `Word`.
-   `#parseHexBytes` interperets a string as a stack of bytes.
-   `#parseByteStack` interperets a string as a stack of bytes, but makes sure to remove the leading "0x".
-   `#parseMap` interperets a JSON key/value object as a map from `Word` to `Word`.

```k
    syntax Word ::= #parseHexWord ( String ) [function]
 // ---------------------------------------------------
    rule #parseHexWord("")   => 0
    rule #parseHexWord("0x") => 0
    rule #parseHexWord(S)    => String2Base(replaceAll(S, "0x", ""), 16) requires (S =/=String "") andBool (S =/=String "0x")

    syntax WordStack ::= #parseHexBytes  ( String ) [function]
                       | #parseByteStack ( String ) [function]
 // ----------------------------------------------------------
    rule #parseByteStack(S) => #parseHexBytes(replaceAll(S, "0x", ""))
    rule #parseHexBytes("") => .WordStack
    rule #parseHexBytes(S)  => #parseHexWord(substrString(S, 0, 2)) : #parseHexBytes(substrString(S, 2, lengthString(S))) requires lengthString(S) >=Int 2

    syntax Map ::= #parseMap ( JSON ) [function]
 // --------------------------------------------
    rule #parseMap( { .JSONList                   } ) => .Map
    rule #parseMap( { _   : (VALUE:String) , REST } ) => #parseMap({ REST })                                                requires #parseHexWord(VALUE) ==K 0
    rule #parseMap( { KEY : (VALUE:String) , REST } ) => #parseMap({ REST }) [ #parseHexWord(KEY) <- #parseHexWord(VALUE) ] requires #parseHexWord(VALUE) =/=K 0
```

Unparsing
---------

We need to interperet a `WordStack` as a `String` again so that we can call `Keccak256` on it from `KRYPTO`.

-   `#unparseByteStack` turns a stack of bytes (as a `WordStack`) into a `String`.
-   `#padByte` ensures that the `String` interperetation of a `Word` is wide enough.

```k
    syntax String ::= #unparseByteStack ( WordStack ) [function]
 // ------------------------------------------------------------
    rule #unparseByteStack( .WordStack )   => ""
    rule #unparseByteStack( (W:Int) : WS ) => #padByte(Base2String((W %Int (2 ^Int 8)), 16)) +String #unparseByteStack(WS)

    syntax String ::= #padByte( String ) [function]
 // -----------------------------------------------
    rule #padByte( S ) => S             requires lengthString(S) ==K 2
    rule #padByte( S ) => "0" +String S requires lengthString(S) ==K 1
endmodule
```
