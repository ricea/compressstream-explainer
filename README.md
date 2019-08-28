# Compression Streams Explained
27 August 2019


## What’s all this then?

The "gzip" compression algorithm is extensively used in the web
platform, but up until now has not been exposed to JavaScript. Since
compression is naturally a streaming process, it is a good match for
the [WHATWG Streams](https://streams.spec.whatwg.org/) API.

CompressStream is used to compress a stream. It accepts ArrayBuffer or
ArrayBufferView chunks, and outputs Uint8Array.

DecompressStream is used to decompress a stream. It accepts
ArrayBuffer or ArrayBufferView chunks, and outputs Uint8Array.

Both APIs satisfy the concept of a [transform
stream](https://streams.spec.whatwg.org/#ts-model) from the WHATWG
Streams Standard.


## Goal

The goal is to provide a JavaScript API for compressing and
decompressing data in the "gzip" format.


## Non-goals

*   Compression formats other than "gzip" will not be supported in the
    first version of the API.
*   Support for synchronous compression.


## Example code

### Compress a stream

```javascript
const compressedReadableStream = inputReadableStream.pipeThrough(new CompressStream());
```

### Compress an ArrayBuffer to a Uint8Array

```javascript
async function compressArrayBuffer(in) {
  const cs = new CompressStream();
  const writer = cs.writable.getWriter();
  writer.write(in);
  writer.close();
  const out = [];
  const reader = cs.readable.getReader();
  let totalSize = 0;
  while (true) {
    const { value, done } = await reader.read();
    if (done)
      break;
    out.push(value);
    totalSize += value.byteLength;
  }
  const concatenated = new Uint8Array(totalSize);
  let offset = 0;
  for (const array of out) {
    concatenated.set(array, offset);
    offset += array.byteLength;
  }
  return concatenated;
}
```

### Decompress a Blob to a Blob

```javascript
async function DecompressBlob(blob) {
  const ds = new DecompressStream();
  const decompressedStream = blob.stream().pipeThrough(ds);
  return await new Response(decompressedStream).blob();
}
```


## End-user benefits

Using this API, web developers can compress data to be uploaded, saving
users time and bandwidth.

As an alternative to this API, it is possible for web developers to bundle
an implementation of a compression algorithm with their app. However, that
would have to be downloaded as part of the app, costing the user time and
bandwidth.


## Considered alternatives

*   Why not simply wrap the zlib API?

    Not all platforms use zlib. Moreover, it is not a web-like API and
    it’s hard to use. Implementing CompressStream for zlib helps us
    use it more easily.

*   Why not support synchronous compression?

    We want to be able to offload the work to another thread, which
    cannot be done with a synchronous API.

*   Why not a non-streaming API?

    Gzip backreferences can span more than one chunk. An API which
    only worked on one buffer at a time could not create
    backreferences between different chunks, and so could not be used
    to implement an efficient streaming API. However, a stream-based
    API can be used to compress a single buffer, so it is more
    flexible.


## References & acknowledgements

Original text by Canon Mukai with contributions from Adam Rice, Domenic
Denicola and Yutaka Hirano.
