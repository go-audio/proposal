# Proposal: Audio buffer interface

Author: Matt Aimonetti

With input from Jaana Burcu Dogan and Francesc Campoy Flores.

Last updated: Jan 2 2017

Discussion at https://golang.org/issue/NNNNN.

## Abstract

This proposal suggests core abstractions allowing audio libraries such as
codecs, effects, analyzers and hardware abstraction layers to communicate using
a shared interface.

## Background

Abstracting audio digital signal processing is quite challenging due the nature
of the formats and usages. In a [previous
proposal](https://github.com/golang/proposal/blob/master/design/13432-mobile-audio.md)
Jaana advocated for an audio decoding and playback interface. Defining such an
interface is extremely challenging and the proposal was eventually withdrawn.
Built upon the many conversations Jaana and I had, here is a different approach
with a much reduced scope. Instead of defining a common interface to decode and
play back audio streams, this proposal focuses on the smallest sharable unit:
the audio buffer. It is our belief that trying to define and enforce a common
interface for all 3rd party audio libraries is the wrong approach and would
probably not lead to a significant adoption. However, we can define the base
unit that can be used as input and/or output so all audio libraries written in
Go can communicate between each other in a common and optimized way. The
technical feasibility of this approach was proven by implementing real life
audio processing libraries. The abstractions presented here are extracted from
those implementations. Both Go idioms and audio DSP specific performance
challenges have been considered.

## Proposal

After surveying most major audio libraries in major programming languages, I
believe we should keep the standard audio package as small as possible but with
one clear goal: allow audio library authors to rely on a common interface to
chain libraries together. Encoders, decoders, analyzers and effects should live
in their own packages but they should all share a common "format". As an
example, such as approach should allow users to decode a file such as an aiff
file, run it through an EQ, a compressor/limiter, extract the FFT output and
compress the stream as an ogg file. This example audio chain might be using 6
different audio libraries, implemented by different individuals or teams. To
make things more complicated, the same effect and analyzer libraries might also
be used in a real time context where the signal come from straight from the
sound card or a Digital Audio Workstation.

At the heart of the proposal is the `audio.Buffer` interface:

```golang
type Buffer interface {
	// PCMFormat is the format of buffer (describing the buffer content/format).
	PCMFormat() *Format
	// NumFrames returns the number of frames contained in the buffer.
	NumFrames() int
	// AsFloatBuffer returns a float 64 buffer from this buffer.
	AsFloatBuffer() *FloatBuffer
	// AsFloat32Buffer returns a float 32 buffer from this buffer.
	AsFloat32Buffer() *Float32Buffer
	// AsIntBuffer returns an int buffer from this buffer.
	AsIntBuffer() *IntBuffer
	// Clone creates a clean clone that can be modified without
	// changing the source buffer.
	Clone() Buffer
}
```

This is the common interface that a library author can choose to use an input or
output. This interface contains context information such as the source format
(`PCMFormat` and `NumFrames`) but also methods allowing the (potentially lossy)
conversion of the underlying data to a different format.

The PCM format information is important to understand the audio samples (data
points). Here is the definition of the audio format type:

```golang
// Format is a high level representation of the underlying data.
type Format struct {
	// NumChannels is the number of channels contained in the data
	NumChannels int
	// SampleRate is the sampling rate in Hz
	SampleRate int
	// BitDepth is the number of bits of data for each sample
	BitDepth int
	// Endianess indicate how the byte order of underlying bytes
	Endianness binary.ByteOrder
}
```

Using the `audio.Buffer` interface everywhere could introduce performance
limitations since each library would have to check or potentially convert the
input data. In the case of real time or parallel server processing, performance
might be a serious concern. That's why this proposal also contains 3
implementations of the `audio.Buffer` interface that can be used as raw
implementation but also as input type.

* `audio.FloatBuffer`, an implementation of the interface using an exposed slice
  of float64 values.
* `audio.Float32Buffer`, a performance optimized implementation of the interface
  using an exposed slice of float32 values.
* `audio.IntBuffer`, an implementation of the interface using an exposed slice
  of ints.

The data structure of all those implementation is similar with a pointer for a
format value and a slice containing the audio data. Here is the `IntBuffer`:

```golang
// IntBuffer is an audio buffer with its PCM data formatted as int.
type IntBuffer struct {
	// Format is the representation of the underlying data format
	Format *Format
	// Data is the buffer PCM data as ints
	Data []int
}
```

As the per the interface definition, each implementation knows how to convert
itself into the other buffers. The reason why those other implementations exist
and are part of the package is because some libraries might only want to deal
with a specific data type. They would therefore expose a specific implementation
as an input and output the the interface type. This is crucial to allow library
authors to avoid paying performance penalties. When performance doesn't matter,
then the input type can be generic. The 3 default implementations should cover
the needs of most, if not all implementors.

Note that the audio data contained in a slice is exported on purpose allowing it
can be mutated and buffers reused.

The implementation in code of this proposal is available
[here](https://github.com/go-audio/audio/blob/master/audio.go)
[`wav`](https://github.com/go-audio/wav) and
[`aiff`](https://github.com/go-audio/aiff) codecs using this proposal are also
[available under the same organization](https://github.com/go-audio). Effect
libraries are also on their way.


## Rationale

The main advantage of this approach is that it keeps the standard library
addition small and without complex logic while allowing the Go ecosystem to
build various tools and libraries using a common audio language. Audio libraries
in other programming languages are often fragmented due to the lack of a common
interface encapsulating the audio data. Successful audio APIs such as
[VST](https://en.wikipedia.org/wiki/Virtual_Studio_Technology) have a very small
surface and give developers a lot of freedom.

I see two main tradeoffs of this approach:

1. Developers using Go might expect more from the standard library and might
want to consume common audio formats like they do for images. This is a
realistic request. The addition of wav and aiff codecs implemented on top of
this proposal prove that it's possible. However, I would suggest to start by a
small audio package without any codecs and let the Go community explore APIs
that make sense in their context. We can always add codecs, effects and tools
later on if there is a concrete need and a common ground when it comes to those
libraries. The [go-audio](https://github.com/go-audio) organization can be a
place for those interested in exploring and discussing audio APIs.

2. Performance can still be a concern. I added a `float32` implementation after
[this
discussion](https://github.com/golang/go/issues/13432#issuecomment-269642926).
It is something I considered in the past, mainly because C APIs supporting
floats usually use 32 bit precision. This might be critical for real time
processing and therefore opted to have it part of the proposal. Note that this
buffer comes with a major drawback, most of the standard library such as the
`math` package, operate on `float64`. If someone were to choose this format,
they would not be able to easily rely on the rest of the standard library (at
least without losing the performance optimization). When it comes to integers,
however I don't suggest to opted to stick to `int` only. It this is a concern, I
would suggest we look at an actual case and evaluate the cost of adding another
implementation.

## Compatibility

`audio` being a new package without external dependencies there should not be
any compatibility concerns.

## Implementation

The implementation is already available as a package
[here](https://github.com/go-audio/audio). If this proposal is accepted, I (Matt
Aimonetti) would be converting the package as standard package and send it for
code review during the 1.9 planning release cycle for a 1.9 integration.

## Open issues

The submitted code will contain 3 sections:

* the interface + implementations
* a list of common audio formats for convenience
* a [list of 6 conversion
  functions](https://github.com/go-audio/audio/blob/master/conv.go) very
  commonly used when doing audio DSP.

While the 6 added conversion functions are extremely useful, they aren't audio
specific and probably should live in a different standard package.