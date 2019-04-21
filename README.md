[![Sourcegraph](https://sourcegraph.com/github.com/ugorji/go/-/badge.svg?v=4)](https://sourcegraph.com/github.com/ugorji/go/-/tree/codec?badge)
[![Build Status](https://travis-ci.org/ugorji/go.svg?branch=master)](https://travis-ci.org/ugorji/go)
[![codecov](https://codecov.io/gh/ugorji/go/branch/master/graph/badge.svg?v=4)](https://codecov.io/gh/ugorji/go)
[![GoDoc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)](http://godoc.org/github.com/ugorji/go/codec)
[![rcard](https://goreportcard.com/badge/github.com/ugorji/go/codec?v=4)](https://goreportcard.com/report/github.com/ugorji/go/codec)
[![License](http://img.shields.io/badge/license-mit-blue.svg?style=flat-square)](https://raw.githubusercontent.com/ugorji/go/master/LICENSE)

# go-codec

This repository contains the `go-codec` library.

To install:

```
go get github.com/ugorji/go/codec
```

# Package Documentation


Package codec provides a High Performance, Feature-Rich Idiomatic Go 1.4+
codec/encoding library for binc, msgpack, cbor, json.

Supported Serialization formats are:

```
    - msgpack: https://github.com/msgpack/msgpack
    - binc:    http://github.com/ugorji/binc
    - cbor:    http://cbor.io http://tools.ietf.org/html/rfc7049
    - json:    http://json.org http://tools.ietf.org/html/rfc7159
    - simple:
```

This package will carefully use 'unsafe' for performance reasons in specific
places. You can build without unsafe use by passing the safe or appengine
tag i.e. 'go install -tags=safe ...'. Note that unsafe is only supported for
the last 3 go sdk versions e.g. current go release is go 1.9, so we support
unsafe use only from go 1.7+ . This is because supporting unsafe requires
knowledge of implementation details.

For detailed usage information, read the primer at
http://ugorji.net/blog/go-codec-primer .

The idiomatic Go support is as seen in other encoding packages in the
standard library (ie json, xml, gob, etc).

Rich Feature Set includes:

```
    - Simple but extremely powerful and feature-rich API
    - Support for go1.4 and above, while selectively using newer APIs for later releases
    - Excellent code coverage ( > 90% )
    - Very High Performance.
      Our extensive benchmarks show us outperforming Gob, Json, Bson, etc by 2-4X.
    - Careful selected use of 'unsafe' for targeted performance gains.
      100% mode exists where 'unsafe' is not used at all.
    - Lock-free (sans mutex) concurrency for scaling to 100's of cores
    - In-place updates during decode, with option to zero value in maps and slices prior to decode
    - Coerce types where appropriate
      e.g. decode an int in the stream into a float, decode numbers from formatted strings, etc
    - Corner Cases:
      Overflows, nil maps/slices, nil values in streams are handled correctly
    - Standard field renaming via tags
    - Support for omitting empty fields during an encoding
    - Encoding from any value and decoding into pointer to any value
      (struct, slice, map, primitives, pointers, interface{}, etc)
    - Extensions to support efficient encoding/decoding of any named types
    - Support encoding.(Binary|Text)(M|Unm)arshaler interfaces
    - Support IsZero() bool to determine if a value is a zero value.
      Analogous to time.Time.IsZero() bool.
    - Decoding without a schema (into a interface{}).
      Includes Options to configure what specific map or slice type to use
      when decoding an encoded list or map into a nil interface{}
    - Mapping a non-interface type to an interface, so we can decode appropriately
      into any interface type with a correctly configured non-interface value.
    - Encode a struct as an array, and decode struct from an array in the data stream
    - Option to encode struct keys as numbers (instead of strings)
      (to support structured streams with fields encoded as numeric codes)
    - Comprehensive support for anonymous fields
    - Fast (no-reflection) encoding/decoding of common maps and slices
    - Code-generation for faster performance.
    - Support binary (e.g. messagepack, cbor) and text (e.g. json) formats
    - Support indefinite-length formats to enable true streaming
      (for formats which support it e.g. json, cbor)
    - Support canonical encoding, where a value is ALWAYS encoded as same sequence of bytes.
      This mostly applies to maps, where iteration order is non-deterministic.
    - NIL in data stream decoded as zero value
    - Never silently skip data when decoding.
      User decides whether to return an error or silently skip data when keys or indexes
      in the data stream do not map to fields in the struct.
    - Detect and error when encoding a cyclic reference (instead of stack overflow shutdown)
    - Encode/Decode from/to chan types (for iterative streaming support)
    - Drop-in replacement for encoding/json. `json:` key in struct tag supported.
    - Provides a RPC Server and Client Codec for net/rpc communication protocol.
    - Handle unique idiosyncrasies of codecs e.g.
      - For messagepack, configure how ambiguities in handling raw bytes are resolved
      - For messagepack, provide rpc server/client codec to support
        msgpack-rpc protocol defined at:
        https://github.com/msgpack-rpc/msgpack-rpc/blob/master/spec.md
```


## Extension Support

Users can register a function to handle the encoding or decoding of their
custom types.

There are no restrictions on what the custom type can be. Some examples:

```go
    type BisSet   []int
    type BitSet64 uint64
    type UUID     string
    type MyStructWithUnexportedFields struct { a int; b bool; c []int; }
    type GifImage struct { ... }
```

As an illustration, MyStructWithUnexportedFields would normally be encoded
as an empty map because it has no exported fields, while UUID would be
encoded as a string. However, with extension support, you can encode any of
these however you like.


## Custom Encoding and Decoding

This package maintains symmetry in the encoding and decoding halfs. We
determine how to encode or decode by walking this decision tree

```
    - is type a codec.Selfer?
    - is there an extension registered for the type?
    - is format binary, and is type a encoding.BinaryMarshaler and BinaryUnmarshaler?
    - is format specifically json, and is type a encoding/json.Marshaler and Unmarshaler?
    - is format text-based, and type an encoding.TextMarshaler and TextUnmarshaler?
    - else we use a pair of functions based on the "kind" of the type e.g. map, slice, int64, etc
```

This symmetry is important to reduce chances of issues happening because the
encoding and decoding sides are out of sync e.g. decoded via very specific
encoding.TextUnmarshaler but encoded via kind-specific generalized mode.

Consequently, if a type only defines one-half of the symmetry (e.g. it
implements UnmarshalJSON() but not MarshalJSON() ), then that type doesn't
satisfy the check and we will continue walking down the decision tree.


## RPC

RPC Client and Server Codecs are implemented, so the codecs can be used with
the standard net/rpc package.


## Usage

The Handle is SAFE for concurrent READ, but NOT SAFE for concurrent
modification.

The Encoder and Decoder are NOT safe for concurrent use.

Consequently, the usage model is basically:

```
    - Create and initialize the Handle before any use.
      Once created, DO NOT modify it.
    - Multiple Encoders or Decoders can now use the Handle concurrently.
      They only read information off the Handle (never write).
    - However, each Encoder or Decoder MUST not be used concurrently
    - To re-use an Encoder/Decoder, call Reset(...) on it first.
      This allows you use state maintained on the Encoder/Decoder.
```

Sample usage model:

```go
    // create and configure Handle
    var (
      bh codec.BincHandle
      mh codec.MsgpackHandle
      ch codec.CborHandle
    )

    mh.MapType = reflect.TypeOf(map[string]interface{}(nil))

    // configure extensions
    // e.g. for msgpack, define functions and enable Time support for tag 1
    // mh.SetExt(reflect.TypeOf(time.Time{}), 1, myExt)

    // create and use decoder/encoder
    var (
      r io.Reader
      w io.Writer
      b []byte
      h = &bh // or mh to use msgpack
    )

    dec = codec.NewDecoder(r, h)
    dec = codec.NewDecoderBytes(b, h)
    err = dec.Decode(&v)

    enc = codec.NewEncoder(w, h)
    enc = codec.NewEncoderBytes(&b, h)
    err = enc.Encode(v)

    //RPC Server
    go func() {
        for {
            conn, err := listener.Accept()
            rpcCodec := codec.GoRpc.ServerCodec(conn, h)
            //OR rpcCodec := codec.MsgpackSpecRpc.ServerCodec(conn, h)
            rpc.ServeCodec(rpcCodec)
        }
    }()

    //RPC Communication (client side)
    conn, err = net.Dial("tcp", "localhost:5555")
    rpcCodec := codec.GoRpc.ClientCodec(conn, h)
    //OR rpcCodec := codec.MsgpackSpecRpc.ClientCodec(conn, h)
    client := rpc.NewClientWithCodec(rpcCodec)
```


## Running Tests

To run tests, use the following:

```
    go test
```

To run the full suite of tests, use the following:

```
    go test -tags alltests -run Suite
```

You can run the tag 'safe' to run tests or build in safe mode. e.g.

```
    go test -tags safe -run Json
    go test -tags "alltests safe" -run Suite
```


## Running Benchmarks

Please see http://github.com/ugorji/go-codec-bench .


## Caveats

Struct fields matching the following are ignored during encoding and
decoding

```
    - struct tag value set to -
    - func, complex numbers, unsafe pointers
    - unexported and not embedded
    - unexported and embedded and not struct kind
    - unexported and embedded pointers (from go1.10)
```

Every other field in a struct will be encoded/decoded.

Embedded fields are encoded as if they exist in the top-level struct, with
some caveats. See Encode documentation.

## Exported Package API

```go
const CborStreamBytes byte = 0x5f ...
const GenVersion = 10
var GoRpc goRpc
var MsgpackSpecRpc msgpackSpecRpc
func GenHelperDecoder(d *Decoder) (gd genHelperDecoder, dd genHelperDecDriver)
func GenHelperEncoder(e *Encoder) (ge genHelperEncoder, ee genHelperEncDriver)
type BasicHandle struct{ ... }
type BincHandle struct{ ... }
type BytesExt interface{ ... }
type CborHandle struct{ ... }
type DecodeOptions struct{ ... }
type Decoder struct{ ... }
    func NewDecoder(r io.Reader, h Handle) *Decoder
    func NewDecoderBytes(in []byte, h Handle) *Decoder
type EncodeOptions struct{ ... }
type Encoder struct{ ... }
    func NewEncoder(w io.Writer, h Handle) *Encoder
    func NewEncoderBytes(out *[]byte, h Handle) *Encoder
type Ext interface{ ... }
type Handle interface{ ... }
type InterfaceExt interface{ ... }
type JsonHandle struct{ ... }
type MapBySlice interface{ ... }
type MissingFielder interface{ ... }
type MsgpackHandle struct{ ... }
type MsgpackSpecRpcMultiArgs []interface{}
type RPCOptions struct{ ... }
type Raw []byte
type RawExt struct{ ... }
type Rpc interface{ ... }
type Selfer interface{ ... }
type SimpleHandle struct{ ... }
type TypeInfos struct{ ... }
    func NewTypeInfos(tags []string) *TypeInfos
```
