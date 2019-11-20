# dicom-fuzz

Fuzzing [DCMTK](https://dicom.offis.de/dcmtk.php.en) with [afl-fuzz](http://lcamtuf.coredump.cx/afl/).

The high level procedure is as follows:

1. Select a target binary
2. Select or generate a small set of testcases
3. Run afl-fuzz for a day or so on $(($(nproc) - 1)) cores
4. Run afl-cov
5. Review coverage output for gaps
6. Goto 2

## File Format

Targeting `dcmdump`.

### Corpus

I'm starting out with a small set of testcases from [go-dicom](https://github.com/grailbio/go-dicom/tree/master/examples). I plan to try out some files from [TCIA](https://www.cancerimagingarchive.net/). I also plan to write some tools to generate my own files that include particular Value Representations and encoding schemes.

### Method

1. Remove `PixelData` and `OverlayData` tags. We don't care about how third-party libraries parse imaging data - we just want to hit the parser. This also significantly reduces the size of the testcases.

```bash
dcmodify -ea PixelData -ea OverlayData $INPUT_TESTCASES/*.dcm
```

2. Minimize

```bash
afl-cmin -i $INPUT_TESTCASES -o $OUTPUT_TESTCASES -- /usr/local/bin/dcmdump @@
```

TODO: Use `afl-tmin` to further reduce the size of the testcases.

3. Fuzz

```bash
afl-fuzz -i $TESTCASES_DIR -o $FINDINGS_DIR -x dicom.dict /usr/local/bin/dcmdump @@
```

`dicom.dict` is included in this repository - it includes the definitions of all known Value Representations. I could include a tag dictionary as well but idk if it will be all that useful.

### Findings

o.O

#### Infinite loop when parsing a malformed DICOMDIR

Still triaging this one - there is some hardcoded autocorrection that causes DCMTK to repeatedly remove bytes from a Directory Record. It's not clear if this has security implications.

#### Suspicious hang when removing spaces from UI tags

## Network Protocol

Targeting several binaries - probably `storescp` and `storescu` to start.

TBD - will need to do some weird instrumentation to get this to work with afl-fuzz.

## Using afl-cov

Make a copy of the `dcmtk` sources and compile with profiling enabled. After the
`project` entry in CMakeLists.txt add the following:

```cmake
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
```

FIXME: I don't think you need `-ftest-coverage` in the linker

Run afl-cov as follows:

```bash
python2 $AFL_COV_BIN -d $FINDINGS_DIR --code-dir $DCMTK_SRC --coverage-cmd $DCMTK_SRC/$BUILDDIR/bin/dcmdump AFL_FILE --lcov-web-all --overwrite
```

FIXME: This takes a LONG TIME holy moly. Multiprocessing support??? Python3???
