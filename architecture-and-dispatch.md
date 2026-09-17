# Architecture and Runtime Dispatch

## DSP-style extraction

When optimizing an operation embedded in a decoder:

1. Extract a small C reference function with a clear internal contract.
2. Store it in an internal context function pointer.
3. Initialize the C implementation first.
4. Let architecture-specific initialization replace it only when the required
   CPU features are available.
5. Test the function directly through checkasm as well as through the decoder.

This keeps architecture selection out of the decode loop and gives checkasm a
stable C-versus-optimized interface.

For the bitpacked decoder, the internal contract is:

```c
/* width must be a positive multiple of 8;
 * src contains width * 5 / 2 bytes. */
void unpack_yuv422p10(const uint8_t *src,
                      uint16_t *y, uint16_t *u, uint16_t *v,
                      int width);
```

The decoder sends aligned multiples of eight through the selected function and
retains the scalar implementation for smaller decoder tails.

## Architecture organization

- Put AArch64 initialization and assembly in `libavcodec/aarch64/` and register
  the objects in that directory's `Makefile`.
- Put x86 initialization and x86inc assembly in `libavcodec/x86/`.
- Keep CPU-feature decisions in the architecture initializer rather than in the
  hot assembly routine.
- Declare architecture entry points only where needed; avoid unrelated header
  and type-style churn while introducing dispatch.

## Endianness is part of dispatch correctness

Selecting an implementation that produces the wrong native-endian pixel format
is a correctness regression, even if it works on every commonly tested host.
An endian guard is an appropriate immediate fallback when an optimized routine
has not been made correct. When support is small and idiomatic, prefer fixing
the assembly and retaining normal SIMD dispatch on both endian modes.

Do not infer endian portability from successful little-endian tests. Inspect the
element types of every load/store and execute a big-endian build when possible.
