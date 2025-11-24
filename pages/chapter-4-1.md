# Chapter 4: Encoding & Evolution

## Agenda - Full Hour Session

### Part 1: Presentation
1. Introduction and aims of the chapter
2. Formats for encoding data
3. Modes of data flow

### Part 2: Discussion
4. Structured Q&A and Group Reflection

---

# Introduction

<div style="margin-top: 20px">
> "Everything changes and nothing stands still."
</div>
<!--
 This chapter makes reference to "evolvability", briefly discussed in chapter 1.
 We should aim to build systems that make it easy to adapt to change.
-->
- changes to application features require changes to the data it stores
- in most cases this cannot happen instantly, especially with huge applications
- new and old versions of the code and the data co-exist at the same time in the system
- **Backward compatibility:** newer code can read data written by older code
- **Forward compatibility:** older code can read data written by newer code

Backward compatibility is easier but forward compatibility is much trickier

---

# Aims of the chapter

<br>
- discuss several data encoding formats (JSON, XML, Protocol Buffers, Thrift & Avro)
- how they handle schema changes
- how they support systems where old and new data or code need to co-exist
- how they are used for data storage and communication in web services, REST, RPC and messaging systems.

---

# Formats for encoding data

We have two different representations of data in programs:

1. in-memory: data is kept in objects, lists, arrays, trees etc.
Optimized for efficient access and manipulated by the CPU (using pointers)

2. writing to a file or over a network: requires encoding into self contained sequence of bytes (eg JSON document).
A pointer won't make sense to any other process since the byte sequence looks different from the in-memory ones.


---

# Translation needed
<br	/>
<img src="../assets/chapter04/encoding-meme.png" alt="meme related to encoding" class="w-100"/>
<br	/>

1. in-memory representation to byte sequence - **encoding** (serialization)
2. byte sequence representation to in-memory - **decoding** (parsing)

There are different libraries and encoding formats to choose from.
---

# Language specific formats of encoding

Language specific encoding formats exists.
> Eg: `java.io.serializable` (Java), Marshal (Ruby), pickle (Python).

Not recommended because:

- language lock-in: integrating with other systems with different language might be difficult or impossible
- security concerns: since the process needs to be able to instantiate arbitrary classes for decoding, an attacker can exploit this to remotely execute malicious code.
- neglect of forward and backward compatibility 
- sometimes bad performance and bloated encoding

> Rule of thumb: unless it's for temporary purposes, don't use your language's built in encoding!

---

# Standardized encoding formats

We also have standardized language agnostic formats of encoding generally categorized into Textual formats (JSON, XML, CSV) and Binary formats.

## Textual formats
- JSON: widely known and supported, mainly due to built-in support in web browsers.
- XML: also widely known and supported but more verbose and unnecessarily complicated.
- CSV: also popular, but less powerful.

Unfortunately, they also have some problems.
---

# Problems with Textual encoding formats
### 1. Ambiguity around encoding of numbers
- XML & CSV does not distinguish between a number and a string of digits, except with an external schema
- JSON does not distinguish between integers and floating-point numbers. It also does not specify a precision.

There is a huge issue in accuracy when dealing with large numbers, eg. integers greater than 2<sup>53</sup>. Decoding in a language that uses floating point numbers, such as JS results in inaccurate number.

> For example, Twitter uses a 64-bit number to represent each tweet. As a workaround, the JSON returned by their API includes tweet ids twice. Once as a JSON number and once as a decimal string.
---

### 2. No support for binary strings
- Although XML and JSON have good support for unicode strings (human readable text), they don't support sequence of bytes with no character encoding.
- Workaround is by encoding binary data as text using Base64, with the schema used to help interpret the value.
- Hacky and bloats data size by 33%

---

### 3. Optional (no) schema support
- Although XML & JSON both support schemas, they are sometimes quite complicated to learn and implement. Applications that don't use schemas need to hardcode the appropriate encoding/decoding logic.
- CSV has no schema, so the application defines what each row and column mean. Changes need to be handled manually.
---

<br/>
<img src="../assets/chapter04/encoding-meme2.png" alt="another meme related to encoding" class="w-100"/>
<br/>

---

# Should we use these textual encoding formats (JSON, XML, CSV)

> YES!

- Still good enough for many purposes, especially as data interchange formats (from one org to another)
- As long as people agree on the format, the efficiency does not matter as much.
---

# Binary encoding formats
They are binary native (handles images or file buffers directly), compact and faster to parse.

- More suitable for very large datasets (TB)
- mostly for internal use within an organization

> JSON variants: BSON, BJSON, BISON, MessagePack
> XML variants: WBXML, Fast Infoset
---

## Example encoding in MessagePack

Original Data: `{"name": "John", "age": 42}`.

Encoded: `82 a4 6e 61 6d 65 a4 4a 6f 68 6e a3 61 67 65 2a`.

#### Encoding Breakdown:
- <b>82:</b> This is the header for a map with two elements.
- <b>a4:</b> This is the header for a string of length 4.
- <b>6e 61 6d 65:</b> This is the UTF-8 encoded byte sequence for the string "name".
- <b>a4:</b> This is the header for a string of length 4.
- <b>4a 6f 68 6e:</b> This is the UTF-8 encoded byte sequence for the string "John".
- <b>a3:</b> This is the header for a string of length 3.
- <b>61 67 65:</b> This is the UTF-8 encoded byte sequence for the string "age".
- <b>2a:</b> This is the header for an integer (specifically, the fixint format for small positive integers), which corresponds to the decimal value 42

> Takes a total of 16 bytes, compared to the JSON equivalent of 24 bytes

---

# Thrift & Protocol Buffers
- Both based on the same principle and require a schema for any data that is encoded. 

> Thrift Interface Definition Language (IDL)
```
struct Person {
	1: required string	name,
	2: optional i32		age
}
```

> Protobuff Interface Definition Language (IDL)
```
message Person {
	required string	name = 	1,
	optional int32	age	=	2;
}
```

---

# How they work
They come with a codegen tool that takes a schema definition and produces classes that implement the schema in various programming languages. So your code can call this generated code to encode/decode the records of the schema.

Thrift offers to binary encoding formats:
- Binary Protocol (uses 14 bytes to encode the same data)
- Compact Protocol (uses 10 bytes to encode the same data)

> Key difference is they use field tags (number aliases) instead of the field names.
> Compact Protocol packs the field type and tag number into a single byte and by using variable length integers. 

- Protobuff encodes very similar to Compact Protocol but uses 6 bytes instead.

---

# Breakdown for Thrift Binary Protocol
Encoded (14 bytes): `0b 00 01 00 00 00 04 4a 6f 68 6e 08 00 02 00 00 00 2a 00`

- Field 1: name (string, ID 1) (8 bytes total)
	- `0b`: Type byte for string (value 11).
	- `00 01`: 2-byte field ID for name (value 1), big-endian.
	- `00 00 00 04`: 4-byte length prefix for the string "John", big-endian signed 32-bit integer.
	- `4a 6f 68 6e`: UTF-8 bytes for "John" (4 bytes).
- Field 2: age (i32, ID 2) (5 bytes total)
	- `08`: Type byte for i32 (value 8).
	-` 00 02`: 2-byte field ID for age (value 2), big-endian.
	- `00 00 00 2a`: 4-byte integer for the value 42, big-endian (hex 2a).
- End of Struct Marker (1 byte)
	- `00`: Stop byte (value 0) to mark the end of the struct. 

---

# Breakdown for Thrift Compact Protocol
Encoded (10 bytes): `5b 04 4a 6f 68 6e 88 54 00`

- Field 1: name (string, ID 1) (5 bytes total)
	- `5b`: A single byte combining the field's type (string, type 11 in compact protocol) and delta from the previous field (delta 1).
	- `04`: VarInt length prefix for the string "John".
	- `4a 6f 68 6e`: UTF-8 bytes for "John" (4 bytes).
- Field 2: age (i32, ID 2) (4 bytes total)
	- `88`: A single byte combining the field's type (i32, type 8 in compact protocol) and delta from the previous field (delta 1).
	- `54`: Zig-zag VarInt for the integer value 42.
- End of Struct Marker (1 byte)
	- `00`: Stop byte to mark the end of the struct.

---

# Breakdown for Protobuff
Encoded (6 bytes): `0a 04 4a 6f 68 6e 10 2a`

- Field 1: name (string, ID 1) (5 bytes total)
	- `0a`: Tag byte. This combines the field ID 1 (shifted left by 3 bits) and the Wire Type 2 (Length-delimited data): (1 << 3) | 2 = 10.
	- `04`: Length VarInt for the string "John".
	- `4a 6f 68 6e`: UTF-8 bytes for "John" (4 bytes).
- Field 2: age (int32, ID 2) (1 byte total)
	- `10`: Tag byte. This combines the field ID 2 and the Wire Type 0 (VarInt data): (2 << 3) | 0 = 16.
	- `2a`: Value VarInt. The integer value 42, which fits in a single byte (hex 2a is 42 decimal).

---

# Final comparison
<b>Format</b>			    <b>Total Bytes</b>
Protocol Buffers			6 bytes
Thrift Compact Protocol		10 bytes
Thrift Binary Protocol		14 bytes
MessagePack					16 bytes
JSON (UTF-8)				24 bytes

---

# Schema evolution
Thrift and Protobuff handle backward and forward compatibility using field tags.

## Forward compatibility
- If a field value is not set, it is simply omitted from the encoded record.
- You can change the name of a field in the schema, but not the field's tag since it makes the encoded data invalid.
- You can add new fields to the schema, with each a new tag number.
- If old code tries to read data written by new code, including a new filed with a tag number it doesn't know, it can simply ignore that field.

> This maintains forward compatibility: old code can read records written by new code.

---

# Schema evolution
Thrift and Protobuff handle backward and forward compatibility using field tags.

## Backward compatibility
- Using unique tag numbers, new code can always read old data.
- If you add a new field, you cannot make it required. Old code would not have written it. So every new field must be optional or have a default value.
- You can only remove a field that is optional
- You can never us the same tag number again

> This maintains backward compatibility: new code can read records written by old code.

---

# Avro binary encoding
Different from Protobuff and Thrift, although also uses a schema.

- Avro IDL (intended for human editing)
```
record Person {
	string name;
	int age;
}
```

- Avro schema (JSON format, machine-readable)
```
{
  "type": "record",
  "name": "Person",
  "fields": [
    { "name": "name", "type": "string" },
    { "name": "age", "type": "int" }
  ]
}
```

---

# Breakdown for Avro

Encoding (6 bytes): `08 4a 6f 68 6e 54`

- Field 1: name (string) (5 bytes total)
	- `08`: This is a VarInt representing the length of the string "John" (4 bytes). In Avro, integers (and lengths) are VarInt-encoded. 08 in hex (8 in decimal) is used for a length of 4 after applying Avro's specific VarInt encoding logic (4 becomes 8 via zig-zag, then 8 is single-byte VarInt).
	- `4a 6f 68 6e`: UTF-8 bytes for "John" (4 bytes).
- Field 2: age (int) (1 byte total)
	- `54`: This is a VarInt representing the integer value 42. In Avro, integers use zig-zag VarInt encoding. The value 42 is encoded into the single byte 54 in hex.

---

# Schema evolution in Avro
> NB: In Avro, data can only be decoded correctly if the code reading the data is using the exact same schema as the code that wrote the data. Any mismatch would mean incorrectly decoded data.

It enforces evolution using:

- Writer's schema: whatever version of the schema used for encoding at the time of writing, that may compiled into the application.
- Reader's schema: whatever schema the application code is relying on to decode the data.

The writer and reader's schema only need to be compatible. The Avro library resolves any differences and translates data from writer's schema into reader's schema.

---

# Schema evolution rules in Avro
- Forward compatibility means you can have a new version of the schema as writer and an old version as a reader.
- Backward compatibility means you can have a new version of the schema as the reader and an old version as writer.
- To maintain compatibility, you may only add or remove a field that has a default value. 
- The default value is used to fill in for the missing field when a reader using the new schema tried to read a record written with the old schema

---

# Benefits of Schemas
- Much more compact than the various "binary JSON" variants, since they can omit field names from encoded data.
- The schema is a valuable form of documentation. You can guarantee that it is up to date since it is needed for decoding.
- Allows you to check for backward/forward compatibility changes, before anything is deployed.
- Can generate code (for statically typed languages) from the schema and useful for type checking at compile time. 

---

# End of first part

<br/>
<img src="../assets/chapter04/encoding-meme3.jpg" alt="end-p4" class="w-150 center m-auto"/>
<br/>