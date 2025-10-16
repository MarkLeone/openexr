# Gdeflate Integration Review Guide

## Overview
- Adds `GDEFLATE` as a supported compression method across the C and C++ APIs, wiring it into enums, name/description tables, and compressor factories.
- Implements page-based gdeflate encode/decode helpers in `OpenEXRCore`.
- Disabled by default.  Uses wrappers around gdeflate entry points in NVIDIA’s libdeflate fork with conditional compilation and error fallback.
- Extends CMake and Bazel build rules to optionally fetch NVIDIA's fork of libdeflate.
- Expands tests to exercise gdeflate, including multi-page compression and decompression.

## How Gdeflate Compression Works
- Channel data is first de-interleaved, consistent with zip compression path (via `internal_zip_deconstruct_bytes()`).
- Gdeflate compresses input buffer as in-place 64 KiB pages (determined via `libdeflate_gdeflate_compress_bound()`), then records each page’s actual compressed length in the metadata array.

```327:341:src/lib/OpenEXRCore/compression.c
for (i = 0; i <= last_page; ++i)
{
    out_pages[i].data = page_data_start + i * out_page_size;
    out_pages[i].nbytes = (i < last_page) ? out_page_size
                                          : page_data_avail - last_page * out_page_size;
}
outsz = wrap_gdeflate_compress (comp, in, in_bytes, out_pages, (size_t) page_count);
```

- After compression succeeds, it writes `[page_count][page_0_size]…[page_{N-1}_size]` in little-endian form and compacts the page data so metadata is immediately followed by the concatenated pages.

```357:377:src/lib/OpenEXRCore/compression.c
metadata    = (uint32_t*) out_base;
metadata[0] = one_from_native32 ((uint32_t) page_count);
for (i = 0; i < page_count; ++i)
{
    metadata[i + 1] = one_from_native32 ((uint32_t) out_pages[i].nbytes);
    total_compressed += out_pages[i].nbytes;
}
for (i = 0; i < page_count; ++i)
{
    memmove (dest, out_pages[i].data, out_pages[i].nbytes);
    dest += out_pages[i].nbytes;
}
```

- After compression, the metadata block captures the number of pages plus each page’s actual compressed length so a final memmove can pack the payload contiguously.

- If compressed output exceeds the original packed size, the encoder falls back to copying the uncompressed buffer so readers never see expansion (just as the zip compressor does).

## Decompression
- Metadata is read first: `page_count` and per-page sizes are loaded, and page pointers are calculated into the contiguous compressed payoad data.
- `wrap_gdeflate_decompress()` receives those `libdeflate_gdeflate_in_page` descriptors, emits a single reconstructed buffer, and the pipeline then re-interleaves via `internal_zip_reconstruct_bytes()`.

## Review Topics



### 1. Pipeline Integration (`compression.c`, `internal_zip.c`)
- Encoder buffer allocation chooses between traditional ZIP sizing and new gdeflate sizing via `exr_compress_gdeflate_max_buffer_size()`.
- Encode/decode helpers reuse ZIP byte shuffling but call `exr_compress_buffer_gdeflate()` / `exr_uncompress_buffer_gdeflate()`; confirm error handling and copy-on-expansion logic match existing expectations.

```556:569:src/lib/OpenEXRCore/compression.c
if (part->comp_type == EXR_COMPRESSION_GDEFLATE)
{
    uint64_t page_count = 0;
    uint64_t page_size  = 0;
    size_t alloc_size = exr_compress_gdeflate_max_buffer_size (
        maxbytes,
        &page_count,
        &page_size);
    rv = internal_encode_alloc_buffer (
        encode,
        EXR_TRANSCODE_BUFFER_COMPRESSED,
        &(encode->compressed_buffer),
        &(encode->compressed_alloc_size),
        alloc_size);
}
```
### 2. CMake Configuration (Enabling Gdeflate)
- By default, `OPENEXR_ENABLE_GDEFLATE` is toggled automatically: if an external libdeflate with gdeflate symbols is found, the flag is set, otherwise it stays off.
- To force the NVIDIA fork from CMake, configure with `-DOPENEXR_FORCE_FETCHCONTENT_DEFLATE=ON` (optionally override `OPENEXR_DEFLATE_REPO` / `OPENEXR_DEFLATE_TAG`); the build will FetchContent the fork, build it as an OBJECT library, and mark gdeflate enabled.
- You can also force the internal vendored library (`-DOPENEXR_FORCE_INTERNAL_DEFLATE=ON`) which keeps gdeflate disabled, or supply your own libdeflate installation that exports the gdeflate API.

```205:333:cmake/OpenEXRSetup.cmake
option(OPENEXR_FORCE_INTERNAL_DEFLATE "Force using an internal libdeflate" OFF)
option(OPENEXR_FORCE_FETCHCONTENT_DEFLATE "Force fetching libdeflate from git repo (NVIDIA fork with gdeflate)" OFF)
set(OPENEXR_DEFLATE_REPO "https://github.com/NVIDIA/libdeflate.git" CACHE STRING "Git repo for FetchContent libdeflate source")
set(OPENEXR_DEFLATE_TAG "gdeflate" CACHE STRING "Git tag/branch for FetchContent libdeflate source")
...
elseif(OPENEXR_FORCE_FETCHCONTENT_DEFLATE)
  # Using FetchContent to get NVIDIA fork
  message(STATUS "Fetching libdeflate from ${OPENEXR_DEFLATE_REPO} @ ${OPENEXR_DEFLATE_TAG}")
  ...
  set(OPENEXR_ENABLE_GDEFLATE ON)
else()
  # Check if external libdeflate has gdeflate support
  include(CheckCSourceCompiles)
  ...
  if(HAVE_LIBDEFLATE_GDEFLATE)
    set(OPENEXR_ENABLE_GDEFLATE ON)
  else()
    set(OPENEXR_ENABLE_GDEFLATE OFF)
  endif()
endif()
```
- In the auto-detect branch, CMake compiles a tiny probe that calls `libdeflate_alloc_gdeflate_compressor()`; success flips `OPENEXR_ENABLE_GDEFLATE` on for externally provided libdeflate builds, otherwise the flag stays off.

### 3. Conditional compilation

- `internal_gdeflate_wrapper.h` centralizes the feature flag: when `OPENEXR_ENABLE_GDEFLATE` is unset it supplies stub structs and returns errors, so the pipeline still compiles but throws if gdeflate entry points are invoked.
- With the flag enabled the wrappers forward straight to NVIDIA’s extensions after wiring the context allocators through `libdeflate_set_memory_allocator()`.
- Bazel mirrors this gating via module overrides (comments in `MODULE.bazel` show how to switch to the NVIDIA fork), and the CMake probe above toggles the same flag automatically for external/system libdeflate.

```97:116:src/lib/OpenEXRCore/internal_gdeflate_wrapper.h
static inline size_t
wrap_gdeflate_compress (
    struct libdeflate_gdeflate_compressor*  comp,
    const void*                             in,
    size_t                                  in_bytes,
    struct libdeflate_gdeflate_out_page*    out_pages,
    size_t                                  out_page_count)
{
#ifdef OPENEXR_ENABLE_GDEFLATE
    return libdeflate_gdeflate_compress (
        comp, in, in_bytes, out_pages, out_page_count);
#else
    (void) comp;
    (void) in;
    (void) in_bytes;
    (void) out_pages;
    (void) out_page_count;
    return 0;
#endif
}
```



### 4. API Surface Updates
- Enum additions in `ImfCompression.h`, `openexr_attr.h`, and `ImfCRgbaFile.h` must remain in sync with lookup tables and `NUM_COMPRESSION_METHODS`.
- `ImfCompressor.cpp` now returns `GdeflateCompressor`; the fallback path throws `NoImplExc` when gdeflate is unavailable.
- C API (`openexr_compression.h`) exports new helpers—verify documentation and naming match existing style.

```348:352:src/lib/OpenEXR/ImfCompressor.cpp
case GDEFLATE_COMPRESSION:
    ret = new GdeflateCompressor (hdr, maxScanLineSize, 16);
    break;
```

### 5. Test Strategy
- Core compression tests bump `IMG_WIDTH`/`IMG_STRIDE_X` to force multi-page chunks under gdeflate; ensure no regressions for other codecs.
- `testGdeflateCompression` executes only when the feature flag is enabled; skip path prints a clear message.
- API-level tests (e.g., `testCompressionApi.cpp`) keep codec lists, lossless assertions, and `NUM_COMPRESSION_METHODS` in sync.

```1667:1673:src/test/OpenEXRCoreTest/compression.cpp
#ifdef OPENEXR_ENABLE_GDEFLATE
    testComp (tempdir, EXR_COMPRESSION_GDEFLATE);
#else
    std::cout << "  gdeflate support not available - test skipped" << std::endl;
#endif
```

## Additional Talking Points
- Chunk decoding depends on metadata parsing—even for zero-sized payloads—so edge cases around empty pages should be sanity-checked.
