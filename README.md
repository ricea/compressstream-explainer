# Compression Streams Explained
27 August 2019


## What’s all this then?

The "gzip" and "deflate" compression algorithms are extensively used in the
web platform, but up until now have not been exposed to JavaScript. Since
compression is naturally a streaming process, it is a good match for the
[WHATWG Streams](https://streams.spec.whatwg.org/) API.

CompressionStream is used to compress a stream. It accepts ArrayBuffer or
ArrayBufferView chunks, and outputs Uint8Array.

DecompressionStream is used to decompress a stream. It accepts
ArrayBuffer or ArrayBufferView chunks, and outputs Uint8Array.

Both APIs satisfy the concept of a [transform
stream](https://streams.spec.whatwg.org/#ts-model) from the WHATWG
Streams Standard.


## Goal

The goal is to provide a JavaScript API for compressing and
decompressing data in the "gzip" or "deflate" formats.


## Non-goals

*   Compression formats other than "gzip" and "deflate" will not be
    supported in the first version of the API.
*   Support for synchronous compression.


## Example code

### Gzip-compress a stream

```javascript
const compressedReadableStream = inputReadableStream.pipeThrough(new CompressionStream('gzip'));
```

### Deflate-compress an ArrayBuffer to a Uint8Array

```javascript
async function compressArrayBuffer(in) {
  const cs = new CompressionStream('deflate');
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

### Gzip-decompress a Blob to a Blob

This treats the input as a gzip file regardless of the mime-type. The output
Blob has an empty mime-type.

```javascript
async function DecompressBlob(blob) {
  const ds = new DecompressionStream('gzip');
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
    it’s hard to use. Implementing CompressionStream for zlib helps us
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

*   Why not support other formats in the first version?

    Gzip and Deflate are ubiquitous and already shipping in every browser.
    This means the incremental cost of exposing them is very low. They are
    used so extensively in the web platform that there is almost zero
    chance of them ever being removed, so committing to supporting them
    long-term is safe.


## Future work

There are a number of possible future expansions which may increase the
utility of the API:

* Other compression algorithms, including "brotli".
* Implementing new compression algorithms in JavaScript or WASM.
* Options for algorithms, such as setting the compression level.
* "Low-latency" mode, where compressed data is flushed at the end of each
  chunk. Currently data is always buffered across chunks. This means many
  small chunks may be passed in before any compressed data is produced.


## References & acknowledgements

Original text by Canon Mukai with contributions from Adam Rice, Domenic
Denicola, Takeshi Yoshino and Yutaka Hirano.
