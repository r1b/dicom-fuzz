# dicom-fuzz

Fuzzing DCMTK with afl-fuzz.

## File Format

Targeting `dcmdump`.

### Corpus

I'm starting out with a small set of testcases from [go-dicom](https://github.com/grailbio/go-dicom/tree/master/examples). I plan to try out some files from [TCIA](https://www.cancerimagingarchive.net/). I also plan to write some tools to generate my own files that include particular Value Representations and encoding schemes.

### Method

1. Remove PixelData and OverlayData. We don't care about how third-party libraries parse imaging data - we just want to hit the parser. This also significantly reduces the size of the testcases.

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

`dicom.dict` is included in this repository - it includes the definitations of all known Value Representations. I could include a tag dictionary as well but idk if it will be all that useful.

### Findings

o.O

#### Infinite loop when parsing a malformed DICOMDIR

Still triaging this one - there is some hardcoded autocorrection that causes DCMTK to repeatedly remove bytes from a Directory Record. It's not clear if this has security implications.

#### Suspicious hang when removing spaces from UI tags

## Network Protocol

Targeting several binaries - probably `storescp` and `storescu` to start.

TBD - will need to do some weird instrumentation to get this to work with afl-fuzz.
