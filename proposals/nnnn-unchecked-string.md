# UncheckedString (raw string support for Swift)

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Alastair Houghton](https://github.com/al45tair)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/pull/NNNNN) or [swiftlang/swift-evolution-staging#NNNNN](https://github.com/swiftlang/swift-evolution-staging/pull/NNNNN)
* Review: ([pitch](https://forums.swift.org/...))

## Summary of changes

This proposal adds an `UncheckedString` type to the Swift standard library
and compiler.  `UncheckedString` assumes no particular character encoding
and is even generic on code unit size, so you can use it to hold 8-bit,
16-bit or even 32-bit string types.

## Motivation

There are plenty of strings that are not necessarily UTF-8, for instance:

* File paths
* Command line arguments
* Environment variables
* HTTP and MIME headers

Many Swift programs that manipulate these kinds of strings currently assume
that they are UTF-8 and will fail with a fatal error when supplied with a
string that is not.  This can easily happen in practice, particularly on
traditional UNIX-like platforms where filenames are really a "bag of bytes"
in no particular encoding, and users are free to configure their terminals
to use any character encoding they wish.

Even on Apple platforms, where people commonly assume UTF-8, the user's
terminal might not be UTF-8 (and they might even be connected using SSH
from some other system, so this can't be fixed by making macOS terminal
only support UTF-8), which means command line arguments and environment
variables could end up being encoded using some other encoding scheme.

Additionally, on Windows, UTF-16/WTF-16/UCS-2 strings are in common use,
but Windows software generally doesn't pay too much attention to validity,
so it's very common to see things that are not really valid UTF-16.  Not
only is the conversion between UTF-16 and UTF-8 Swift `String` inefficient,
but it very commonly will not actually work because the string wasn't
really valid Unicode.  Using UTF-16 strings on Windows is also awkward
because of the lack of a literal syntax.

Currently, programs that expect to deal correctly with all of these cases
are forced to either use unnatural representations for the string data in
question (like `Array<UInt8>` or `Data`), which makes inspecting and
manipulating the string awkward, or they end up building their own raw
string type, which will often have poor performance and a limited API
surface.  It's also difficult or impossible to write string literals in
code when taking either of these approaches.

`UncheckedString` solves all of these problems by creating a new string-like
type that supports

* Arbitrary `FixedWidthInteger` code units.
* A similar set of APIs to `String`.
* Efficient COW storage, with the small string optimization.
* Compiler-supported string literal syntax, including a new hexadecimal
  escape sequence to allow the use of raw code units.
* `encode()`/`decode()` APIs to transform between `UncheckedString`
  and `String`.

All with good performance.

## Proposed solution

The proposal adds a new _generic_ string-like type, `UncheckedString`,
along with `UncheckedSubString` and `UncheckedStringProtocol`:

```swift
struct UncheckedString<Element: FixedWidthInteger>: UncheckedStringProtocol {
  ...
}

struct UncheckedSubString<Element: FixedWidthInteger>: UncheckedStringProtocol {
  ...
}

protocol UncheckedStringProtocol: 
    BidirectionalCollection, Equatable, Hashable, Comparable,
    CustomDebugStringConvertible, CustomUncheckedStringConvertible
  where Iterator.Element: FixedWidthInteger,
    Index == Int,
    SubSequence: UncheckedStringProtocol,
    UncheckedStringElement == Element {
  ...
}
```

The API for these types is similar to the API on `String`, `SubString` and
`StringProtocol`.  Significant departures include:

* `UncheckedString` is generic over its element type, which is not
  `Character`, but instead some `FixedWidthInteger`.
* `UncheckedString.Index` is `Int`.  This makes sense because subscripting
  provides direct access to the code units themselves, and we do not know
  the encoding.
* `UncheckedString` is encoding-agnostic.  It's just a collection of code
  units.  It needn't even be ASCII-based, though in practice most uses of
  it will be.
* `UncheckedString` supports an extra character escape, `\x{hh}`, which
  allows the direct specification of individual code units.
* There is presently no regular expression support for `UncheckedString`.

There are also counterparts of `CustomStringConvertible`,
`ExpressibleByStringLiteral`, and `ExpressibleByStringInterpolation`
that support the use of literals and even interpolation with
`UncheckedString`.

## Detailed design

### `UncheckedString` literals

`UncheckedString` supports literals through the
`ExpressibleByUncheckedStringLiteral`
and `ExpressibleByUncheckedStringInterpolation` protocols:

```swift
extension UncheckedString: ExpressibleByUncheckedStringLiteral {
  public typealias UncheckedStringLiteralType = UncheckedString<Element>

  @_transparent
  public init(uncheckedStringLiteral value: UncheckedString<Element>) {
    self = value
  }
}

extension UncheckedString: ExpressibleByUncheckedStringInterpolation {
  public init(stringInterpolation: StringInterpolation) {
    self.init(taking: stringInterpolation.chars)
  }
}
```

Currently valid string literals will continue to be interpreted as `String`
or `Character` literals as they are today, for instance:

```swift
let name = "René Descartes"  // `name` is a `String`
```

Users can explicitly request an `UncheckedString`

```swift
let utf8Name: UncheckedString<UInt8> = "René Descartes"
let utf16Name: UncheckedString<UInt16> = "René Descartes"
let utf8Name2 = "René Descartes" as UncheckedString<UInt8>
let utf16Name2 = "René Descartes" as UncheckedString<UInt16>
```

Swift source files are encoded in UTF-8, and any Unicode codepoints used
in a string literal will be transformed into appropriately encoded Unicode
given the width of the `UncheckedString` element type (so
`UncheckedString<UInt8>` would get UTF-8 encoding, `UncheckedString<UInt16>`
would get UTF-16 encoding, and `UncheckedString<UInt32>` will get UCS-4
encoding for any explicit Unicode in the input).  The existing `\u{hh}`
escape sequences are also supported for `UncheckedString`, with the same
semantics as literal characters.

We are also adding an `\x{hh}` escape sequence that allows the user to
specify directly a code unit in an `UncheckedString`.  This escape is _only_
permitted for `UncheckedString` literals, and the compiler is aware that if
it sees it, the literal in question _must_ be an `UncheckedString` and not
a `String` or `Character` literal, for instance:

```swift
let iso8859Name = "Ren\x{e9} Descartes" // UncheckedString<UInt8>

// This is a compile-time error:
let invalidName: String = "Ren\x{e9} Descartes"
```

The compiler further guarantees that `UncheckedString` literal data is
encoded in the binary as a sequence of code units of the specified type,
using the endianness appropriate for the target.  That is,

```swift
let wideString: UncheckedString<UInt16> = "Hello, World 😀"
```

on a little-endian system results in the data

```
48 00 65 00 6c 00 6c 00 6f 00 2c 00 20 00 57 00   H.e.l.l.o.,. .W.
6f 00 72 00 6c 00 64 00 20 00 3d d8 00 de         o.r.l.d. .=Ø.Þ
```

in the final binary.  It is not transcoded at runtime, but instead forms
a real 16-bit string constant.

### Compile-time literal type resolution

The compiler currently hard-codes knowledge about the types of string
literals, but we need it to choose appropriately on the basis of the
literal in question (if it contains a `\x{hh}` escape) and/or the
surrounding context, since we want things like

```swift
let utf8Name: UncheckedString<UInt8> = "René Descartes"
let utf16Name: UncheckedString<UInt16> = "René Descartes"
let utf8Name2 = "René Descartes" as UncheckedString<UInt8>
let utf16Name2 = "René Descartes" as UncheckedString<UInt16>
```

or even

```swift
let nameExpression: UncheckedString<UInt8> = "René" + " " + "Descartes"
```

to work.

To accomplish this, we add a family of "umbrella" protocols such as
`ExpressibleByPossiblyUncheckedStringLiteral`, re-parenting the existing
`ExpressibleByStringLiteral` protocol and also acting as the parent of a
new `ExpressibleByUncheckedStringLiteral` protocol.  This allows us to
express in the type system the idea that a literal _might_ be a normal
`String` literal, but _also_ might be an `UncheckedString` literal; the
existing type checking machinery can then pick the most appropriate
interpretation based on surrounding type information.

While doing this, we ensure that in the absence of any other information:

1. A literal that does not contain `\x{hh}` will be a `String` literal, and
2. A literal that _does_ contain `\x{hh}` will be an
   `UncheckedString<UInt8>` literal.

`UncheckedString` literals must also worry about their element type; to
account for this, we modify the compiler so that it maintains UTF-8 data,
as well as a "splice" list containing any characters specified using
`\x{hh}` escapes.  This is then used by SILGen and IRGen to generate an
appropriate constant on the basis of the type information, which by the
time it gets to those layers has been fixed.

### `CustomUncheckedStringConvertible` API

`UncheckedString` has a counterpart to `CustomStringConvertible`, namely
`CustomUncheckedStringConvertible`, which allows a type to declare that
it can be converted to an `UncheckedString`:

```swift
/// A type that can provide a raw, native-width representation of itself for
/// interpolation into an `UncheckedString`.
///
/// Unlike `String`'s `CustomStringConvertible`, which produces Unicode text,
/// a `CustomUncheckedStringConvertible` type produces raw `Element` data
/// directly -- no implicit encoding, transcoding, or textual description is
/// involved. `UncheckedString` interpolation only accepts values that
/// conform to this protocol (rather than any `CustomStringConvertible`
/// value, the way `String` interpolation does), since nothing about
/// `UncheckedString` implies a text encoding that an arbitrary describable
/// value could be rendered through.
///
/// `UncheckedStringProtocol` (and therefore `UncheckedString` and
/// `UncheckedSubString`) conforms to this protocol automatically, producing
/// its own character data. Any other type can opt in by declaring the
/// conformance and implementing `withUncheckedStringRepresentation`.
@available(SwiftStdlib 9999, *)
public protocol CustomUncheckedStringConvertible {
  /// The element type of the raw representation this type provides.
  associatedtype UncheckedStringElement: FixedWidthInteger

  /// Calls the given closure with a buffer containing this value's raw
  /// `UncheckedStringElement` representation.
  func withUncheckedStringRepresentation<R, Failure>(
    _ body: (Span<UncheckedStringElement>) throws(Failure) -> R
  ) throws(Failure) -> R
}
```

### `UncheckedStringProtocol` API

Again paralleling `String`, `UncheckedString` and `UncheckedSubString` both
implement `UncheckedStringProtocol`:

```swift
// A type that can represent a string as a collection of characters.
///
/// Unlike `StringProtocol`, no assumptions are made about the encoding or
/// type of the characters.
@available(SwiftStdlib 9999, *)
public protocol UncheckedStringProtocol
  : BidirectionalCollection, Equatable, Hashable, Comparable,
    CustomDebugStringConvertible, CustomUncheckedStringConvertible
  where Iterator.Element: FixedWidthInteger,
    Index == Int,
    SubSequence: UncheckedStringProtocol,
    UncheckedStringElement == Element
{
  typealias SubSequence = UncheckedSubString<Element>

  /// Calls the given closure with a buffer of `Element`s,
  /// which are *not* necessarily NUL-terminated.
  func withCharacterData<R, E>(
    _ body: (Span<Element>) throws(E) -> R
  ) throws(E) -> R
}
```

### `UncheckedString` API

`UncheckedString` has a single property, `count`, and can be constructed
empty, or from a `Collection`.

```swift
/// A string value that is a collection of characters in an unspecified
/// encoding.
...
@available(SwiftStdlib 9999, *)
public struct UncheckedString<E: FixedWidthInteger>: UncheckedStringProtocol {
  public typealias Element = E
  public typealias SubSequence = UncheckedSubString<Element>
  public typealias Index = Int

  @inlinable
  public var count: Int { get }

  /// Constructs an empty string
  @inlinable
  public init()

  /// Constructs a string from a collection of character elements.
  ///
  /// - Parameters:
  ///
  ///   - c: A `Collection` containing the character elements.
  ///
  public init<C: Collection>(_ c: C) where C.Element == Element
}
```

It also has support for C strings:

```swift
@available(SwiftStdlib 9999, *)
extension UncheckedString {
  /// Creates a string from a NUL-terminated sequence of characters.
  @inlinable
  public init(cString: UnsafePointer<Element>)

  /// Calls the given closure with a pointer to the contents of the string,
  /// represented as a NUL-terminated sequence of `Element`s.
  @inlinable
  public func withCString<R, Failure>(
    _ body: (UnsafePointer<Element>) throws(Failure) -> R
  ) throws(Failure) -> R
}
```

along with special support for `CChar` for `UncheckedString<UInt8>`:

```swift
// For UInt8, also allow CChar
@available(SwiftStdlib 9999, *)
extension UncheckedString where Element == UInt8 {
  /// Calls the given closure with a pointer to the contents of the string,
  /// represented as a NUL-terminated sequence of `Element`s.
  @inlinable
  public func withCString<R, Failure>(
    _ body: (UnsafePointer<CChar>) throws(Failure) -> R
  ) throws(Failure) -> R

  /// Creates a string from a NUL-terminated sequence of characters.
  @inlinable
  public init(cString nullTerminatedCharacters: UnsafePointer<CChar>)
}
```

An additional feature of `UncheckedString` is support for _immortal_ strings,
which are strings whose memory is allocated elsewhere somehow (for instance
by the compiler or linker, or by code outside of the `UncheckedString`
implementation) and that will never be released.

```swift
@available(SwiftStdlib 9999, *)
extension UncheckedString {
  /// Creates a string from a NUL-terminated immortal string.
  @inlinable
  public init(immortalString: UnsafePointer<Element>)

  // Creates a string from an immortal string that isn't NUL terminated.
  @inlinable
  public init(immortalString: UnsafeBufferPointer<Element>)
}
```

`UncheckedString` also provides direct access to character data using the
`withCharacterData` API:

```swift
@available(SwiftStdlib 9999, *)
extension UncheckedString {
  /// Calls the given closure with a buffer of `Element`s,
  /// which are *not* necessarily NUL-terminated.
  @inlinable
  public func withCharacterData<R, Failure>(
    _ body: (Span<Element>) throws(Failure) -> R
  ) throws(Failure) -> R
}
```

Finally, sub-sequences use the `UncheckedSubString` type:

```swift
@available(SwiftStdlib 9999, *)
public struct UncheckedSubString<E: FixedWidthInteger>
  : UncheckedStringProtocol
{
  public typealias Element = E
  public typealias SubSequence = UncheckedSubString<Element>
  public typealias Index = Int

  ...
}
```

### Encoding/Decoding

As a convenience for working with encoded strings, `UncheckedString`
provides a `decode` API that attempts to convert its contents to a UTF-8
`String`:

```swift
public enum UncheckedStringFailEncoding {
  case fail
}

public enum UncheckedStringSubstituteEncoding {
  case substitute
}

@available(SwiftStdlib 9999, *)
public extension UncheckedString {
  /// Attempt to decode an `UncheckedString` using the specified encoding
  ///
  /// - Parameters:
  ///   - encoding:  The encoding to use.
  ///   - onInvalidEncoding: `.fail` to return `nil`, or `.substitute` to
  ///                replace invalid encodings with the substitution
  ///                character.
  ///
  /// - Returns a `String` if decoding was successful.
  func decode<Encoding: Unicode.Encoding>(
    as encoding: Encoding.Type,
    onInvalidEncoding: UncheckedStringFailEncoding = .fail
  ) -> String? where Encoding.CodeUnit == Element

  /// Attempt to decode an `UncheckedString` using the specified encoding
  ///
  /// - Parameters:
  ///   - encoding:  The encoding to use.
  ///   - onInvalidEncoding: `.fail` to return `nil`, or `.substitute` to
  ///                replace invalid encodings with the substitution
  ///                character.
  ///
  /// - Returns a `String` if decoding was successful.
  func decode<Encoding: Unicode.Encoding>(
    as encoding: Encoding.Type,
    onInvalidEncoding: UncheckedStringSubstituteEncoding
  ) -> String where Encoding.CodeUnit == Element
}
```

We also add an `encode` API, mirroring this one, on `String` itself:

```swift
@available(SwiftStdlib 9999, *)
public extension String {
  /// Attempt to encode a `String` using the specified encoding.
  ///
  /// - Parameters:
  ///   - encoding:  The encoding to use.
  ///   - onUnsupportedEncoding: `.fail` to return `nil`, or `.substitute`
  ///                to use a substitution character.
  ///
  /// - Returns an `UncheckedString` if encoding was successful.
  func encode<Encoding: Unicode.Encoding>(
    as encoding: Encoding.Type,
    onUnsupportedEncoding: UncheckedStringFailEncoding = .fail
  ) -> UncheckedString<Encoding.CodeUnit>?

  /// Attempt to encode a `String` using the specified encoding.
  ///
  /// - Parameters:
  ///   - encoding:  The encoding to use.
  ///   - onUnsupportedEncoding: `.fail` to return `nil`, or `.substitute`
  ///                to use a substitution character.
  ///
  /// - Returns an `UncheckedString` if encoding was successful.
  func encode<Encoding: Unicode.Encoding>(
    as encoding: Encoding.Type,
    onUnsupportedEncoding: UncheckedStringSubstituteEncoding
  ) -> UncheckedString<Encoding.CodeUnit>
}
```

This allows the user to write things like

```swift
guard let ansiName = "René Descartes".encode(as: Windows1252.self) else {
  fatalError("Cannot encode René Descartes' name for Windows")
}

guard let swiftName = ansiName.decode(as: Windows1252.self) else {
  fatalError("Cannot decode René Descartes' name from Windows")
}

assert(swiftName == "René Descartes")
```

## Conversion to C string types

`UncheckedString` supports conversions to C string pointers in a similar
manner to the conversions supported by `String`.  That is, it is permitted
to pass an `UncheckedString` to a function that expects an
`UnsafePointer<Element>` or `UnsafeRawPointer`.

This has the same risks as the similar feature of the `String` type; the
`UncheckedString` must persist at least long enough for the code in question
to make use of the pointer it has been passed.  It is up to the user to
ensure that that is the case.

This feature is a considerably quality of life improvement on Windows, where
interacting with the wide character APIs is commonplace, as it allows users
to make explicit use of UTF-16 strings and string constants without having
to transcode at runtime.

## Implementation Details

It's worth noting a few implementation details here; `UncheckedString` does
have the small string optimization, and under the covers stores its data
using `UncheckedStringStorage`:

```swift
@frozen
@usableFromInline
enum UncheckedStringStorage<CharType: FixedWidthInteger> {
  case empty
  case small(SmallUncheckedStringStorage<CharType>)
  case immortal(ImmortalUncheckedStringStorage<CharType>)
  case `dynamic`(DynamicUncheckedStringStorage<CharType>)

  @usableFromInline
  var count: Int {
    switch self {
      case .empty: return 0
      case .small(let rawStorage): return Int(rawStorage.count)
      case .immortal(let rawStorage): return Int(rawStorage.count)
      case .dynamic(let rawStorage): return Int(rawStorage.count)
    }
  }
}
```

The `small` variant stores character data inline, and is 16 bytes on 64-bit
platforms or 8 bytes on 32-bit.

The `immortal` variant holds an immutable pointer, a count and some flags.

The `dynamic` variant stores the actual data in an array of `[CharType]`,
and also holds a count (on 64-bit) and some flags.

The only important flag currently is a NUL-termination flag.  Dynamic strings
are _always_ NUL-terminated.  Immortal strings _may_ be NUL-terminated, or
they may not.  Small strings do not have the flag and are not NUL-terminated.

The implementation will promote a string to `dynamic` if it grows above the
size allowed for a small string, or if space is reserved for a string too
large for a small string.  Once promoted to dynamic, it will not be
demoted to `small`, to avoid allocation thrashing — if a string is being
mutated and is `dynamic`, it's likely it will be mutated further, and we
don't want to throw away the backing store at that point.

### Performance

Performance of `UncheckedString<UInt8>` is generally better than that of
`String`, since, of course, `UncheckedString` does not need to do any
Unicode processing.  In some cases (`count`, for instance), it is very
significantly faster; in others (`hasPrefix`, `hasSuffix`), it is on par,
but in most cases it's 2 to 5 times faster than `String`.

That is not to say that people should use `UncheckedString` instead of
`String`; for most programs, `String` will remain the right choice, as it
is safer and supports Unicode and indeed regular expressions, and there
are tricks (such as working on the UTF-8 view) that give good performance
for `String` and that should probably be preferred in most cases.

## Source compatibility

This proposal is purely additive.  It does affect parsing slightly, in
that a string literal containing a `\x{hh}` escape is _only_ allowed for
`UncheckedString` literals, but since `\x{hh}` is not currently a valid
escape sequence there should be no source compatibility issue.

Making the literals work did involve some surgery in the type checker,
but this should not affect anyone not using the new type.

The changes in the standard library are purely additive in nature, and
won't affect anything not using these types.

## ABI compatibility

This proposal _does_ introduce new umbrella protocols as parents of
the `ExpressibleByStringLiteral`, `ExpressibleByStringInterpolation`,
`ExpressibleByExtendedGraphemeClusterLiteral` and
`ExpressibleByUnicodeScalarLiteral`, in order to allow the type checker
to correctly select between `String` and `UncheckedString` literals.

This should not impact existing code, however.

## Implications on adoption

This proposal requires new versions of Swift Syntax and the compiler,
in addition to the standard library.

Programs using the `UncheckedString` type will need to be using a new
enough compiler and standard library.

## Future directions

We could, in future, add a `WideString` type that holds _checked_
UTF-16 data.  This might be interesting on Windows, or perhaps for
Java interop, though in both of those cases there might be concerns
about whether other code on the system actually cared to maintain
proper UTF-16 invariants (i.e. it's possible that `UncheckedString<UInt16>`
or `UncheckedString<CWideChar>` is preferable anyway).

`FilePath` could use `UncheckedString` as its backing store.

We could add regular expression support for `UncheckedString`s.

## Alternatives considered

### Storing non-UTF-8 string data in `Array` or `Data`.

The downsides here are significant; there is no literal support,
whatsoever, and `Array` and `Data` do not support all of the methods
one might expect on a `String`-like type, which leads to rather
unnatural looking programs.

Additionally, `UncheckedString` has a `CustomDebugStringConvertible`
override, which means that you can easily inspect one in a string-like
format, which isn't the case for `Array` or `Data`.

### Attempting to convert non-UTF-8 data to UTF-8 `String`s

This works _some_ of the time, for instance for string data that we know
to be pure ASCII, but is potentially inefficient — for instance, there is
no need to go checking MIME or HTTP headers to make sure that the header
names are valid UTF-8, and if you do so then you also have to handle the
situation where one of them turns out not to be (for instance because an
attacker is sending malformed data to your program).

Additionally, conversions can be expensive, and quite often you would need
to convert back the other way again when serializing data.

## Acknowledgments

None yet :-)
