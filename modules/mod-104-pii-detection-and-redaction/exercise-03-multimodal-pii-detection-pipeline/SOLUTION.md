# Exercise 03 — Multimodal PII Detection Pipeline — Reference Solution

## 1. Solution overview

This is an **implementation exercise**. The learner is asked to build one
pipeline that detects and redacts PII across more than one modality — at a
minimum, text and images, and ideally also structured documents (PDFs) and
audio. The point is not to call one API three times; it is to design the
*shape* of a cross-modal detector so that the audit trail, the taxonomy,
and the operator semantics are consistent across modalities.

The assessable output is a repository with:

1. A `pipeline/` package exposing a single `detect(asset) -> Findings`
   entry point that dispatches on a declared `media_type`
   (`text/plain`, `image/png`, `image/jpeg`, `application/pdf`, `audio/wav`).
2. Per-modality adapters (text, image via OCR, PDF via layout parse, audio
   via ASR) that each produce findings in **one shared schema**.
3. A redactor that applies the appropriate operation per modality — in-text
   replacement, pixel blur / black-box, PDF content-stream redaction, audio
   silencing or beeping.
4. A `fixtures/` directory with small samples in each modality and
   corresponding gold annotations.
5. A metrics script that reports per-modality precision / recall / F1 and
   wall-clock budget.

The reference below walks through the shared `Finding` schema, an adapter
per modality, the trickiest cross-cutting failure modes (coordinate frames,
reading order, timestamp drift), and the rubric used to grade a submission.

## 2. Worked implementation

### 2.1 Shared `Finding` schema

Every modality emits the same record shape. Downstream components (audit
log, redactor, metrics) consume findings without knowing which modality
produced them.

```python
# schema.py
from dataclasses import dataclass, field
from typing import Optional, List, Literal


@dataclass
class TextLocator:
    start: int
    end: int


@dataclass
class ImageLocator:
    # pixel-space bounding box of the containing image or page render
    x: int
    y: int
    w: int
    h: int
    page: Optional[int] = None   # for PDFs
    rotation: Optional[int] = 0  # EXIF-adjusted


@dataclass
class AudioLocator:
    start_ms: int
    end_ms: int
    channel: Optional[int] = None


@dataclass
class Finding:
    entity_type: str          # e.g., "EMAIL_ADDRESS" from the shared taxonomy
    score: float              # [0, 1]
    recognizer: str           # name of the recognizer that fired
    text: Optional[TextLocator] = None
    image: Optional[ImageLocator] = None
    audio: Optional[AudioLocator] = None
    excerpt: Optional[str] = None   # the string/phrase (NEVER the raw media)
    notes: List[str] = field(default_factory=list)


@dataclass
class Findings:
    media_type: Literal["text/plain", "image/png", "image/jpeg",
                        "application/pdf", "audio/wav"]
    asset_id: str
    findings: List[Finding]
```

Design notes:

- A single finding can have *all three* locators populated — e.g., a PDF
  page where the OCR engine recovered character offsets and a bounding
  box, or an ASR transcript with both a word index and a millisecond span.
  Downstream always reads the locator that matches the planned redaction.
- `excerpt` is the string the detector saw (e.g., "john@example.com"),
  never the raw media bytes. This matters when findings are sent to an
  audit store with a different retention class.
- `recognizer` is required. "Finding came from `ocr+presidio`" is a
  different operational signal than "finding came from `aws-rekognition`".

### 2.2 Text adapter

Thin wrapper around the exercise-01 Presidio pipeline. Converts
`RecognizerResult` into `Finding` objects.

```python
# pipeline/text.py
from .schema import Finding, Findings, TextLocator
from presidio_analyzer import AnalyzerEngine


def detect_text(analyzer: AnalyzerEngine, asset_id: str, text: str,
                score_threshold: float = 0.5) -> Findings:
    results = analyzer.analyze(text=text, language="en",
                               score_threshold=score_threshold)
    out = [
        Finding(
            entity_type=r.entity_type,
            score=r.score,
            recognizer=r.analysis_explanation.recognizer if r.analysis_explanation else "presidio",
            text=TextLocator(start=r.start, end=r.end),
            excerpt=text[r.start:r.end],
        )
        for r in results
    ]
    return Findings(media_type="text/plain", asset_id=asset_id, findings=out)
```

### 2.3 Image adapter (OCR → text detector → projection)

Two choices here. Pick one explicitly per deployment.

- **Option A — OCR-then-detect.** Run OCR (Tesseract, AWS Textract, GCP
  Document AI) to recover text with per-word bounding boxes, run the text
  detector on the concatenated text, then project each text span back to
  the union of its contributing word boxes. This is the right default for
  documents photographed or scanned — the text model you already have is
  the state of the art.
- **Option B — Dedicated vision model.** Call a vision model that
  recognises PII directly (e.g., face detection for images containing
  people, licence-plate detection). This is additive, not a replacement
  for A; vision models are needed for entities that are not text at all
  (faces, signatures, licence plates, hand-written IDs).

The reference uses Option A for text-bearing images and composes Option B
when faces or plates are in scope.

```python
# pipeline/image.py
from typing import List
import pytesseract
from PIL import Image, ImageOps
from .schema import Finding, Findings, ImageLocator
from .text import detect_text


def _ocr_with_boxes(img: Image.Image):
    """Return list of (word, x, y, w, h, start_offset) and the joined text."""
    img = ImageOps.exif_transpose(img)
    data = pytesseract.image_to_data(
        img, output_type=pytesseract.Output.DICT, lang="eng"
    )
    tokens = []
    joined = []
    cursor = 0
    for i, word in enumerate(data["text"]):
        if not word.strip():
            continue
        tokens.append({
            "word": word,
            "x": int(data["left"][i]),
            "y": int(data["top"][i]),
            "w": int(data["width"][i]),
            "h": int(data["height"][i]),
            "start": cursor,
            "end": cursor + len(word),
        })
        joined.append(word)
        cursor += len(word) + 1   # space between words
    return tokens, " ".join(joined)


def detect_image(analyzer, asset_id: str, img: Image.Image) -> Findings:
    tokens, text = _ocr_with_boxes(img)
    text_findings = detect_text(analyzer, asset_id, text).findings
    out: List[Finding] = []
    for f in text_findings:
        overlapping = [t for t in tokens
                       if not (t["end"] <= f.text.start or
                               t["start"] >= f.text.end)]
        if not overlapping:
            continue
        x0 = min(t["x"] for t in overlapping)
        y0 = min(t["y"] for t in overlapping)
        x1 = max(t["x"] + t["w"] for t in overlapping)
        y1 = max(t["y"] + t["h"] for t in overlapping)
        f.image = ImageLocator(x=x0, y=y0, w=x1 - x0, h=y1 - y0)
        out.append(f)
    return Findings(media_type="image/png", asset_id=asset_id, findings=out)
```

Three non-obvious choices:

1. **`ImageOps.exif_transpose` before OCR.** A phone-photographed ID
   oriented portrait on-screen is stored landscape in pixels, with a
   rotation tag. Without the transpose, OCR returns gibberish or
   mis-oriented boxes.
2. **Union of overlapping boxes, not first-match.** A detected
   `PHONE_NUMBER` often spans two OCR tokens (`(555)` + `123-4567`). The
   projection must take the union of every token whose text-offset range
   overlaps the detected span.
3. **Do not redact outside the detected union.** It is tempting to blur
   the whole line. That leaks the location of the redaction (adversary
   knows which line had a phone number) and risks blurring non-PII
   content. Blur only the union.

### 2.4 PDF adapter (layout parse, per-page)

PDFs are a stack of images (rasterised pages) over structured content
streams. The adapter must handle both.

```python
# pipeline/pdf.py
import fitz   # PyMuPDF
from PIL import Image
import io
from .schema import Findings, Finding, ImageLocator
from .image import detect_image


def detect_pdf(analyzer, asset_id: str, pdf_bytes: bytes,
               render_dpi: int = 200) -> Findings:
    doc = fitz.open(stream=pdf_bytes, filetype="pdf")
    all_findings = []
    for page_no in range(len(doc)):
        page = doc[page_no]
        text = page.get_text("text")
        # Path 1: structured text layer
        text_findings = detect_image.__globals__["detect_text"](
            analyzer, f"{asset_id}#p{page_no}:text", text
        ).findings
        for f in text_findings:
            # Project text offsets to PDF rectangles via get_text("dict")
            for rect in _rects_for_offsets(page, f.text.start, f.text.end):
                f.image = ImageLocator(
                    x=int(rect.x0), y=int(rect.y0),
                    w=int(rect.x1 - rect.x0),
                    h=int(rect.y1 - rect.y0),
                    page=page_no,
                )
                all_findings.append(f)
        # Path 2: rasterise and OCR (catches scanned pages)
        pix = page.get_pixmap(dpi=render_dpi)
        img = Image.open(io.BytesIO(pix.tobytes("png")))
        ocr_findings = detect_image(
            analyzer, f"{asset_id}#p{page_no}:ocr", img
        ).findings
        for f in ocr_findings:
            if f.image is not None:
                f.image.page = page_no
            all_findings.append(f)
    return Findings(media_type="application/pdf",
                    asset_id=asset_id, findings=all_findings)
```

Rely on two passes per page — a structured-text pass and a rasterised-OCR
pass. A PDF with a selectable text layer is cheap to analyse structurally;
a scanned PDF needs OCR. Many real PDFs are mixed (text overlay on a
scanned image) and both passes contribute.

### 2.5 Audio adapter

```python
# pipeline/audio.py
from typing import List
from .schema import Findings, Finding, AudioLocator
from .text import detect_text


def detect_audio(analyzer, asset_id: str, asr_segments: List[dict]) -> Findings:
    """asr_segments: list of {"text": "...", "start_ms": int, "end_ms": int}
    produced by whichever ASR you use (Whisper, AWS Transcribe, GCP STT).
    """
    joined = []
    index = []   # char_start -> segment
    cursor = 0
    for seg in asr_segments:
        joined.append(seg["text"])
        index.append((cursor, cursor + len(seg["text"]), seg))
        cursor += len(seg["text"]) + 1
    text = " ".join(joined)
    text_findings = detect_text(analyzer, asset_id, text).findings
    out: List[Finding] = []
    for f in text_findings:
        overlapping = [seg for s, e, seg in index
                       if not (e <= f.text.start or s >= f.text.end)]
        if not overlapping:
            continue
        start_ms = min(s["start_ms"] for s in overlapping)
        end_ms = max(s["end_ms"] for s in overlapping)
        f.audio = AudioLocator(start_ms=start_ms, end_ms=end_ms)
        out.append(f)
    return Findings(media_type="audio/wav", asset_id=asset_id, findings=out)
```

The audio adapter's hardest bug is **ASR word drift**. The transcript
timestamp refers to the recognised word, which may lead or lag the spoken
word by hundreds of milliseconds depending on the model and silence
handling. The redactor MUST pad the beeped region by a configurable
`pad_ms` margin (typical 150–300 ms on either side). Document the padding
and include it in the audit log.

### 2.6 Redactor

```python
# pipeline/redact.py
from PIL import Image, ImageDraw, ImageFilter
from pydub import AudioSegment
from pydub.generators import Sine
from .schema import Findings


def redact_text(text: str, findings: Findings) -> str:
    # Walk findings in reverse offset order so earlier offsets stay valid.
    sorted_f = sorted(
        [f for f in findings.findings if f.text],
        key=lambda f: f.text.start, reverse=True,
    )
    out = text
    for f in sorted_f:
        out = out[:f.text.start] + f"<{f.entity_type}>" + out[f.text.end:]
    return out


def redact_image(img: Image.Image, findings: Findings,
                 mode: str = "box") -> Image.Image:
    draw = ImageDraw.Draw(img)
    for f in findings.findings:
        if not f.image:
            continue
        box = (f.image.x, f.image.y,
               f.image.x + f.image.w, f.image.y + f.image.h)
        if mode == "box":
            draw.rectangle(box, fill="black")
        elif mode == "blur":
            region = img.crop(box).filter(ImageFilter.GaussianBlur(radius=20))
            img.paste(region, box)
    return img


def redact_audio(audio: AudioSegment, findings: Findings,
                 pad_ms: int = 200) -> AudioSegment:
    beep = Sine(1000)
    out = audio
    for f in findings.findings:
        if not f.audio:
            continue
        start = max(0, f.audio.start_ms - pad_ms)
        end = min(len(out), f.audio.end_ms + pad_ms)
        beep_seg = beep.to_audio_segment(duration=end - start,
                                         volume=-6)
        out = out[:start] + beep_seg + out[end:]
    return out
```

PDF redaction has an extra wrinkle: redacting the *rendered image*
without redacting the underlying text stream leaks the data on any
viewer that honours the text layer. Use `page.add_redact_annot` +
`page.apply_redactions()` (PyMuPDF) or an equivalent native API — do not
just paint a black box.

### 2.7 Dispatcher

```python
# pipeline/__init__.py
from .text import detect_text
from .image import detect_image
from .pdf import detect_pdf
from .audio import detect_audio


def detect(analyzer, asset_id: str, media_type: str, payload) -> "Findings":
    if media_type == "text/plain":
        return detect_text(analyzer, asset_id, payload)
    if media_type in {"image/png", "image/jpeg"}:
        return detect_image(analyzer, asset_id, payload)
    if media_type == "application/pdf":
        return detect_pdf(analyzer, asset_id, payload)
    if media_type == "audio/wav":
        return detect_audio(analyzer, asset_id, payload)
    raise ValueError(f"unsupported media_type: {media_type}")
```

### 2.8 Rationale: non-obvious design choices

1. **Shared taxonomy across modalities.** Resist the temptation to define
   `IMAGE_FACE`, `AUDIO_NAME`, `TEXT_PERSON`. One taxonomy (`PERSON`,
   `EMAIL_ADDRESS`, `FACE`, `LICENSE_PLATE`, …) with a locator per
   modality keeps downstream consumers simple.
2. **The adapter owns normalisation, the detector does not.** The image
   adapter handles EXIF rotation; the audio adapter handles ASR drift
   padding; the PDF adapter handles the two-pass problem. Normalising
   inside the detector would couple modality concerns into the text
   analyzer.
3. **Audit every modality identically.** Downstream consumers of the
   `Findings` log should not need a modality switch. Record locator, score,
   recogniser, entity type for each finding regardless of source.
4. **Redaction is destructive; detection must fail closed.** A text
   finding with missing offsets, an image finding with an out-of-bounds
   box, an audio finding with zero duration: each is a bug. Reject the
   finding (do not redact) and alert.

## 3. Validation steps

1. **Round-trip smoke test.** For each modality, run `detect` on a fixture
   that contains a known `EMAIL_ADDRESS` and verify the finding carries
   the correct locator (offsets for text, box for image, ms range for
   audio, page+box for PDF).
2. **Projection correctness.** On the image fixture, run the redactor in
   `box` mode and assert that OCR on the redacted image does not return
   the redacted entity. This is the end-to-end test that catches
   coordinate-frame bugs.
3. **EXIF robustness.** Include a rotated-phone-photo fixture in the image
   tests. The pipeline must detect the same entities on the rotated and
   non-rotated versions.
4. **Audio padding.** Include an ASR transcript in which the timestamp is
   shifted by 150 ms relative to the ground-truth word onset. Assert that
   the redactor still covers the full word.
5. **PDF text-vs-image parity.** Use a PDF with the same content once as a
   text layer and once as a flattened image. Findings must match within
   the mapped taxonomy.
6. **Audit log completeness.** Every detection produces a log record with
   `asset_id`, `media_type`, `entity_type`, `recognizer`, `score`, and the
   relevant locator. Random sample 1% in CI; fail if any record is
   missing a field.
7. **Per-modality metrics.** The metrics harness reports precision,
   recall, and F1 per `(media_type, entity_type)` pair. Text metrics can
   be near 1.0 while image metrics are much lower; reporting only the
   average hides this.

## 4. Rubric

Score out of 10. Below 7 → rework.

| Criterion | Points | Notes |
|---|---|---|
| Single `detect` entry point dispatches on `media_type` | 1 | Not one function per caller |
| Shared `Finding` schema with modality-specific locators | 2 | Duplicate per-modality schemas fail |
| Image path uses OCR with per-word boxes and projects spans onto box union | 1 | Full-line redaction fails |
| PDF path handles both text-layer and rasterised pages | 1 | Scanned-only or text-only fails |
| Audio path pads redaction by configurable `pad_ms` | 1 | Silent cut without padding is a leak |
| Redactor fails closed on malformed findings | 1 | Silent skipping is a leak |
| Metrics reported per `(media_type, entity_type)` | 1 | Aggregate-only fails |
| Audit log includes modality, locator, recognizer, score per finding | 1 | Partial logs fail |
| EXIF / rotation / coordinate-frame correctness demonstrated by test | 1 | Phone-photographed ID is the standard test |

Automatic failures:
- PDF "redaction" that only draws a black box in the rasterised render
  without removing content-stream text.
- Image redactor that blurs full lines rather than the detected box union.
- Audio redactor that silences `[start_ms, end_ms]` without padding,
  leaving audible fragments.
- Shared schema that duplicates entity types per modality (e.g.,
  `TEXT_EMAIL` vs `IMAGE_EMAIL`).

## 5. Common mistakes

- **Forgetting EXIF orientation.** Phone photos carry a rotation tag; OCR
  on the untransposed pixel buffer returns zero text.
- **Redacting only the rendered PDF, not the content stream.** The text is
  still selectable in any viewer that reads the underlying stream — the
  black box is cosmetic.
- **Trusting ASR timestamps to the millisecond.** Word onsets drift
  depending on the ASR model and silence handling; without a padding
  margin, the redactor clips the beginning or end of the word.
- **Returning the raw media in findings.** Keep the finding small: offsets,
  boxes, and a short excerpt. Shipping the audio clip or the image tile
  through the audit store moves PII into a system that was not provisioned
  for it.
- **Running the vision / ASR model inside the request path.** Every
  modality model needs an isolated warm worker pool; invoking them lazily
  blows the latency budget on the first request.
- **One taxonomy per modality.** This forces every downstream consumer to
  know all N modalities. One taxonomy, N locators.
- **Scoring on combined metrics only.** Text F1 of 0.95 can mask image F1
  of 0.4; both stakeholders — text and image — need their own number.

## 6. References

- Microsoft Presidio (text analyzer and anonymizer).
  <https://microsoft.github.io/presidio/>
- Microsoft Presidio image redactor documentation.
  <https://microsoft.github.io/presidio/image-redactor/>
- PyMuPDF (`fitz`) documentation — redaction annotations and
  `apply_redactions`.
  <https://pymupdf.readthedocs.io/>
- Tesseract OCR documentation.
  <https://tesseract-ocr.github.io/tessdoc/>
- OpenAI Whisper (open-source ASR) model card and documentation.
  <https://github.com/openai/whisper>
- AWS Textract and Comprehend PII documentation.
  <https://docs.aws.amazon.com/textract/> and
  <https://docs.aws.amazon.com/comprehend/latest/dg/pii.html>
- Google Cloud Document AI and Sensitive Data Protection.
  <https://cloud.google.com/document-ai/docs> and
  <https://cloud.google.com/sensitive-data-protection/docs>
- NIST AI Risk Management Framework (AI RMF 1.0).
  <https://www.nist.gov/itl/ai-risk-management-framework>
- OWASP Machine Learning Security Top 10 — relevant to the supply-chain
  exposure introduced by third-party vision / ASR models used in the
  adapters.
  <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — tactics applicable to attacks against detectors (e.g.,
  evasion by content embedded in images). <https://atlas.mitre.org/>
