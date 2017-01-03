# Proposal: Audio buffer interface

Author: Matt Aimonetti

With input from Jaana Burcu Dogan and Francesc Campoy Flores.

Last updated: Jan 2 2017

Discussion at https://golang.org/issue/NNNNN.

## Abstract

This proposal suggests core abstractions allowing audio libraries such
as codecs, effects, analyzers and hardware abstraction layers to communicate
using a shared interface.

## Background

Abstracting audio digital signal processing is quite challenging due the nature of the formats and
usages. In a [previous proposal](https://github.com/golang/proposal/blob/master/design/13432-mobile-audio.md) Jaana
advocated for an audio decoding and playback interface. But defining such an interface
is extremely challenging and the proposal was withdrawn.
This proposal is built on the many conversations Jaana and I had which resulted in a different approach and a reduced scope.
Instead of defining a common interface to decode and encode audio streams this proposal
focuses on the smallest shared unit: the audio buffer.
It is our conclusion that trying to define and enforce a common interface for all
3rd party audio libraries is the wrong approach and would probably not lead to a large adoption.
However, we can define the base unit that can be used as input and/or output so all
audio libraries written in Go can communicate between each other in a common and optimized way.
This approach was proven by implementing real life audio processing libraries and extracting
a series of abstractions. Both Go idioms and audio DSP specific performance
challenges have been considered.

## Proposal

After surveying most major audio libraries in Go, Python, Ruby, JS, C, Objective-C and Java I believe
we should keep the standard audio package as small as possible but with one clear goal:
allow audio library authors to rely on a common interface to chain libraries together.
Encoders, decoders, analyzers and effects should live in their own packages but they should
all share a common "format" allowing a user to decode a file such as an aiff file, run it through
an EQ, a compressor/limiter, extract the FFT output and compress the stream as an ogg file.
This example audio chain might be using 6 different audio libraries, implemented by different teams.
To make things more complicated, the same effect and analyzer libraries might also be used in a real time
context where the signal from straight from the sound card.

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

This is the common interface that a library author can choose to use an input or output.
This interface contains context information such as the source format (`PCMFormat` and `NumFrames`)
but also methods allowing the (potentially lossy) conversion of the underlying data to a different
format.

However, using this interface everywhere has some serious performance limitations since each library would
have to check or convert the input data. That's why this proposal also contains 3 implementations
of the `audio.Buffer` interface:

* `audio.FloatBuffer`, an implementation of the interface using an exposed slice of float64 values.
* `audio.Float32Buffer`, a performance optimized implementation of the interface using an exposed slice of float32 values.
* `audio.IntBuffer`, an implementation of the interface using an exposed slice of ints.

The data structure of all those implementation is similar with a pointer for a format value and a slice containing
the audio data. Here is the `IntBuffer`:

```golang
// IntBuffer is an audio buffer with its PCM data formatted as int.
type IntBuffer struct {
	// Format is the representation of the underlying data format
	Format *Format
	// Data is the buffer PCM data as ints
	Data []int
}
```

As the per the interface definition, each implementation knows how to convert itself into the other buffers.
The reason why those other implementations exist and are part of the package is because some libraries
might only want to deal with a specific data type. They would therefore expose a specific implementation
as an input and output the the interface type. This is crucial to allow library authors to avoid paying
great performance penalties. When performance doesn't matter, then the input type can be generic.
The 3 default implementation should cover the needs of most if not all implementors.

Note that the audio data contained in a slice is exported on purpose, so it can be mutated and buffers be reused.

The implementation in code of this proposal is available [here](https://github.com/go-audio/audio/blob/master/audio.go)
[`wav`](https://github.com/go-audio/wav) and [`aiff`](https://github.com/go-audio/aiff) codecs using this proposal
are also [available under the same organization](https://github.com/go-audio). Effect libraries are also on their way.


## Rationale

[A discussion of alternate approaches and the trade offs, advantages, and disadvantages of the specified approach.]

## Compatibility

[A discussion of the change with regard to the
[compatibility guidelines](https://golang.org/doc/go1compat).]

## Implementation

[A description of the steps in the implementation, who will do them, and when.
This should include a discussion of how the work fits into [Go's release cycle](https://golang.org/wiki/Go-Release-Cycle).]

## Open issues (if applicable)

[A discussion of issues relating to this proposal for which the author does not
know the solution. This section may be omitted if there are none.]
