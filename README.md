## Encode and decode structured data to/from a binary buffer

```js
import { BufWriter } from 'nanobuf'

const buf = new BufWriter()

buf.u32(12345) // u̲nsigned ̲3̲2-bit integer
buf.str('Hello world!') // ̲st̲r̲ing
buf.f64(420.69) // ̲6̲4-bit f̲loat (double)

const packet = buf.toUint8Array() // Uint8Array(25) [ 0, 0, 48, 57, 12, ... ]

someSocket.send(packet)
```
```js
// Receiving peer
import { BufReader } from 'nanobuf'

someSocket.on('message', data => {
	const buf = new BufReader(data)
	console.assert(buf.u32() == 12345
	            && buf.str() == 'Hello world!'
	            && buf.f64() == 420.69)
	// Check that we have read everything (optional)
	console.assert(buf.left == 0)
})

```

Perks:
- Dependency-less
- Performant. Know a faster implementation? Open an issue on our github!
- Lightweight (15KB .js)
- Import `index.js` as-is in the browser
- Common format: buffers can be decoded easily with other libraries on other machines (no need to worry about endianness, alignment, integer sizes, etc...)

## Schemas

```js
import { BufWriter, Struct, f32, Arr, bool, Enum } from 'nanobuf'

const Point = Struct({x: f32, y: f32})
const Polyline = Struct({
	points: Arr(Point),
	closes: bool,
	shape: Enum(['straight', 'quadratic-bezier', 'cubic-bezier', 'arc'])
})

const buf = new BufWriter()
buf.encode(Polyline, {
	points: [{x: 1, y: 2}, {x: 5, y: 2}, {x: 3, y: 5}],
	closes: true,
	shape: 'straight'
})
buf.toUint8Array() // Uint8Array(27) [ ... ]

const buf2 = new BufReader(buf)
// Decode
console.log(buf2.decode(Polyline)) // {points: [...], closes: true, shape: 'straight'}
/* Or decode in-place
  const shape = {points: [], closes: false, shape: ''}
  buf2.decode(Polyline, shape)
  console.log(shape)
*/
```
```js
class Point{
	x = 0; y = 0
	static encode(buf, point){
		buf.f32(point.x)
		buf.f32(point.y)
	}
	static decode(buf, point = new Point()){
		point.x = buf.f32()
		point.y = buf.f32()
		return point
	}
	// ...
	static Triangle = Arr(Point, 3)
}

// ...
buf.encode(Point.Triangle, [
	new Point(1, 2),
	new Point(3, 4),
	new Point(5, 6)
])
// Data is always packed tightly
console.assert(buf.byteLength == f32.size * 2 * 3)
// You can also check the size of aggregate types (such as Struct() and Arr() with fixed length)
console.assert(Point.Triangle.size == 24)
```

## Interface
```ts
type StructType = Struct | Arr | Enum | Optional | Padding | bool | u8 | i8 | u16 | i16 | u32 | i32 | u64 | i64 | bu64 | bi64 | f32 | f64 | v16 | v32 | v64 | bv64 | u8arr | str

// Always use buf.<thing>(123) instead of buf.encode(<thing>, 123) when available, since the argument types are static it will always be more performant
class BufWriter{
	bool(n: boolean) // true or false, 1 byte
	u8(n: number) // Write a value in [0, 255], 1 byte
	i8(n: number) // Write a value in [-128, 127], 1 byte
	u16(n: number) // Write a value in [0, 65535], 2 bytes
	i16(n: number) // Write a value in [-32768, 32767], 2 bytes
	u32(n: number) // Write a value in [0, 4294967295], 4 bytes
	i32(n: number) // Write a value in [-2147483648, 2147483647], 4 bytes
	u64(n: number) // Write a value in [0, 18446744073709552000], 8 bytes
	i64(n: number) // Write a value in [-9223372036854776000, 9223372036854776000], 8 bytes
	bu64(n: bigint) // Write a value in [0n,18446744073709551615n], 8 bytes
	bi64(n: bigint) // Write a value in [-9223372036854775808n, 9223372036854775807n], 8 bytes
	f32(n: number) // Write a value in [-3.40282e+38, 3.40282e+38], Precision ~7 digits, 4 bytes
	f64(n: number) // Write a value in [-1.797693134862e+308, 1.797693134862e+308], Precision ~15 digits, 8 bytes
	v16(n: number) // Write a value in [0, 32767], 1 or 2 bytes (depending on largeness of n)
	v32(n: number) // Write a value in [0, 2147483647], 1, 2 or 4 bytes
	v64(n: number) // Write a value in [0, 9223372036854775807], 1, 2, 4 or 8 bytes
	bv64(n: bigint) // Write a value in [0n, 9223372036854775807n], 1, 2, 4 or 8 bytes
	u8arr(arr: Uint8Array, len?: number) // Write the length of arr using v32, followed by arr's raw bytes. If len is specified, the length is implicit and not written (however it must match arr.length)
	str(s: string) // Write s's length using v32, followed by s encoded as UTF-8, maximum s.length*3 bytes
	enum(enumType: Enum, value: string) // Write value's corresponding integer value according to enumType using v32
	encode(type: StructType, value: any) // Encode an arbitrary type
	skip(bytes: number) // Skip a number of bytes, leaving them unwritten (0)

	toUint8Array(): Uint8Array // Get a Uint8Array pointing to the written data. This is a reference and not a copy. Use Uint8Array.slice() to turn it into a copy
	toBuffer(): Buffer // Like toUint8Array() but for node.js Buffers, may not work in the browser
	toReader(): BufReader // Get a BufReader pointing to the written data, with the reading head at the beginning. This is a reference and not a copy
	clone(): BufWriter // Get an actual copy of the written data, as a second BufWriter
	written: number // How many bytes have been written to this buffer so far
	buffer: ArrayBuffer // The underlying array buffer that is being modified. May be larger than this.written (this is intentional to avoid excessive reallocations)
	byteOffset: number // Always 0
	byteLength: number // Same as .written
}

class BufReader extends DataView{
	bool(): boolean // true or false, 1 byte
	u8(): number // Read a value in [0, 255], 1 byte
	i8(): number // Read a value in [-128, 127], 1 byte
	u16(): number // Read a value in [0, 65535], 2 bytes
	i16(): number // Read a value in [-32768, 32767], 2 bytes
	u32(): number // Read a value in [0, 4294967295], 4 bytes
	i32(): number // Read a value in [-2147483648, 2147483647], 4 bytes
	u64(): number // Read a value in [0, 18446744073709552000], 8 bytes
	i64(): number // Read a value in [-9223372036854776000, 9223372036854776000], 8 bytes
	bu64(): bigint // Read a value in [0n,18446744073709551615n], 8 bytes
	bi64(): bigint // Read a value in [-9223372036854775808n, 9223372036854775807n], 8 bytes
	f32(): number // Read a value in [-3.40282e+38, 3.40282e+38], Precision ~7 digits, 4 bytes
	f64(): number // Read a value in [-1.797693134862e+308, 1.797693134862e+308], Precision ~15 digits, 8 bytes
	v16(): number // Read a value in [0, 32767], 1 or 2 bytes (depending on largeness of n)
	v32(): number // Read a value in [0, 2147483647], 1, 2 or 4 bytes
	v64(): number // Read a value in [0, 9223372036854775807], 1, 2, 4 or 8 bytes
	bv64(): bigint // Read a value in [0n, 9223372036854775807n], 1, 2, 4 or 8 bytes
	u8arr(len?: number): Uint8Array // Read a Uint8Array of length len, or (this.v32()) if unspecified
	str(): string // Read a string of length (this.v32()), decoded from UTF-8, maximum result.length*3 bytes
	enum(enumType: Enum): string // Read a v32 and return its corresponding string value according to enumType
	decode(type: StructType): any // Decode an arbitrary type
	skip(bytes: number) // Skip a number of bytes, not bothering to read them
	view(bytes: number) // Skip a number of bytes and return a Uint8Array pointing to those skipped bytes. Similar to u8arr(len)

	toUint8Array(): Uint8Array // Get a Uint8Array pointing to remaining unread data. This is a reference and not a copy. Use Uint8Array.slice() to turn it into a copy
	toBuffer(): Buffer // Like toUint8Array() but for node.js Buffers, may not work in the browser
	toWriter(): BufReader // Get a BufWriter containing a copy of the data that has been read so far, allowing you to continue writing as if the read values had actually been written
	read: number // How many bytes have been read from this buffer so far
	left: number // How many more bytes can be read before reaching the end of the buffer
	overran: boolean // Whether we have reached and passed the end of the buffer. All read functions will return "null" values (i.e, 0, "", Uint8Array(0)[], false, ...)
}

// The following types are constructors and implement the static encode and decode methods, so they can be used with schemas. These methods are ommitted for brevity. Do not use new with these constructors
// E.g u8(257) == 1
// E.g bv64("999") == 999n
// E.g new f64(0) throws TypeError
// E.g i64.encode(someBuf, 123456)
// E.g myNum = f32.decode(someBuf)

type bool

type u8
type i8
type u16
type i16
type u32
type i32
type u64
type i64
type bu64
type bi64

type f32
type f64

// Values 0-127 encoded as:
// 0xxxxxxx
// Values 128-32767 encoded as:  (it's possible to also encode 0-127 like this, forming an overlong encoding, which is valid but undesired)
// 1xxxxxxx xxxxxxxx
type v16
// Values 0-63 encoded as:
// 00xxxxxx
// Values 128-16383 encoded as:
// 01xxxxxx xxxxxxxx
// Values 16384-2147483647 encoded as:
// 1xxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx
type v32
// Values 0-63 encoded as:
// 00xxxxxx
// Values 128-8191 encoded as:
// 010xxxxx xxxxxxxx
// Values 8192-536870911 encoded as:
// 011xxxxx xxxxxxxx xxxxxxxx xxxxxxxx
// Values 536870912-9223372036854775807 encoded as:
// 1xxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx
type v64
type bv64

type u8arr
type str

// Returns a Schema type whose encode/decode is equivalent to calling .u8arr() with the specified length. The length is therefore implicit and not written to or read from buffers
function u8arr.len(length: number): SchemaType

// Returns a Schema type that encodes/decodes an object of the specified structure. Optionally places the static encode and decode functions on a specified class (in that case the class is returned)
function Struct(obj: {[key: string]: SchemaType}, class?: any): SchemaType

// Returns a Schema type that encodes/decodes an array of the specified element type. If len is specified, the length becomes implicit and is not written to or read from buffers
function Arr(type: SchemaType, len?: number): SchemaType

// Returns a Schema type that encodes/decodes a string as a v32, based on a set of specified possible values
// E.g Enum(["apple", "orange", "banana"])
// E.g Enum({apple: 0, orange: 1, banana: 2, pear: 5})
// Invalid values are encoded to an invalid enum and decoded back to the empty string ""
function Enum(values: string[] | {[key: string]: number}): SchemaType

// Returns a Schema type that can encode/decode the type or null. This is done by encoding an extra boolean value immediately before, indicating if the value is null (false) or not (true)
function Optional(type: SchemaType): SchemaType

// Returns a Schema type that advances a number of bytes without interpreting them, essentially skipping the bytes. The encoded/decoded value is always undefined
function Padding(bytes: number): SchemaType
```