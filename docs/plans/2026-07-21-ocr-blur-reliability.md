# OCR and Blur Reliability Implementation Plan

> **For agentic workers:** Implement this plan task-by-task — one task per commit, run each task's test before moving on. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make all four Redact modes work in packaged Eddy: OCR works without a separately installed Tesseract, and video Blur/OCR use the displayed frame instead of the current black placeholder.

**Architecture:** Keep the existing `RedactItem`, `OcrRunner`, and ffmpeg export paths. Resolve an app-local Tesseract before falling back to `PATH`; for video, feed the current `QVideoSink` frame into the existing redaction/OCR objects and pass static blur rectangles to ffmpeg before applying the existing annotation overlay. OCR and redaction rectangles remain fixed in scene coordinates for the whole clip; motion tracking is out of scope.

**Tech Stack:** C++20, Qt 6 Widgets/Multimedia/Test, Tesseract 5.4.0 Windows runtime, ffmpeg.

## Global Constraints

- Do not replace `redactBlur()` or add another blur library; image Blur already passes its rendering and pixel tests.
- Preserve Qt 6.4 source compatibility; `QVideoFrame::toImage()` and `QGraphicsVideoItem::videoSink()` are available on the supported path.
- Keep Linux behavior: explicit OCR options first, app-local Windows runtime second, `PATH`/system tessdata last.
- Bundle only `deu.traineddata` and `osd.traineddata`, matching Eddy's existing `ocr_lang=deu` default.
- Video OCR detects the currently displayed frame once and applies those fixed rectangles to the full clip.
- Video blur strength stays equivalent to the existing strong redaction intent; leave one numeric radius constant beside the ffmpeg filter for later calibration.
- No tracking, keyframes, OCR-on-every-frame, plugin interface, or new redaction hierarchy.

---

## Investigation Findings

- The relevant image tests pass on the current `v1.0.2` tree: `test_items`, `test_raster`, `test_ocr`, `test_redactocrcontroller`, `test_toolcontroller`, and `test_editorwindow`.
- The downloaded `Eddy-1.0.2-windows-x64-setup.exe` contains `eddy.exe` and Qt runtime files but no `tesseract.exe` or `.traineddata` files. `OcrRunner` starts the bare program name `tesseract`, so OCR requires an undocumented system install.
- The real OCR smoke tests call `QSKIP` when Tesseract or the requested language is absent. The Windows release job therefore succeeds without testing OCR.
- `toolBackgroundFor()` returns an all-black `QImage` for every video. Blur samples that black image and OCR scans it, explaining why Blur and both OCR modes fail on video while Blacken still works.
- `writeVideoWithOverlay()` only overlays a static PNG. A transparent overlay cannot blur underlying moving pixels, so export needs an ffmpeg blur stage rather than another `QImage` cache tweak.

## File Map

- Modify `src/ocr.h`, `src/ocr.cpp`: resolve and run the packaged OCR runtime.
- Modify `tests/test_ocr.cpp`, `tests/test_redactocrcontroller.cpp`: cover runtime selection and fail instead of skipping the release-critical smoke path.
- Modify `packaging/windows/build-msi.ps1`, `packaging/windows/build-nsis.ps1`, `.github/workflows/release.yml`: stage and validate Tesseract plus German data.
- Modify `src/items/redactitem.h`, `src/items/redactitem.cpp`: accept a new video-frame source and report blur rectangles in scene coordinates.
- Modify `src/toolcontroller.h`, `src/toolcontroller.cpp`, `src/redactocrcontroller.h`, `src/redactocrcontroller.cpp`: update the source used by future redactions and OCR requests.
- Modify `src/editorwindow.h`, `src/editorwindow.cpp`: consume current video frames, refresh redaction sources, and exclude blur items from the static video overlay.
- Modify `src/videoexporter.h`, `src/videoexporter.cpp`: apply per-region ffmpeg blur before the existing overlay.
- Modify `tests/test_items.cpp`, `tests/test_toolcontroller.cpp`, `tests/test_editorwindow.cpp`, `tests/test_videoexporter.cpp`: lock the image, preview, OCR, and exported-video behavior.
- Modify `README.md`: document packaged OCR and static video redaction semantics.

### Task 1: Lock the working image Blur path

**Files:**
- Modify: `tests/test_editorwindow.cpp`

**Interfaces:**
- Consumes: `ToolController::begin()`, `ToolController::finish()`, `EditorWindow::exportComposite()`.
- Produces: an end-to-end regression proving image Blur still works through the real editor composition path.

- [ ] **Step 1: Add the end-to-end image Blur regression**

Add a test that creates a checkerboard `QImage`, builds `EditorWindow` with animations disabled, draws a Redact item through the window's `ToolController`, exports the composite, and compares pixels:

```cpp
void imageBlurWorksThroughEditorExport() {
    QImage bg(96, 64, QImage::Format_ARGB32_Premultiplied);
    for (int y = 0; y < bg.height(); ++y)
        for (int x = 0; x < bg.width(); ++x)
            bg.setPixelColor(x, y, ((x / 2 + y / 2) % 2) ? Qt::white : Qt::black);
    Config cfg; cfg.animations = false;
    EditorWindow window(bg, cfg, {});
    auto *tools = window.findChild<ToolController *>();
    QVERIFY(tools);
    tools->setTool(ToolType::Redact);
    tools->begin({16, 12});
    tools->finish({80, 52});

    const QImage out = window.exportComposite();
    QCOMPARE(out.pixelColor(4, 4), bg.pixelColor(4, 4));
    QVERIFY(out.pixelColor(48, 32) != bg.pixelColor(48, 32));
    QVERIFY(qAbs(out.pixelColor(48, 32).red() - out.pixelColor(49, 32).red()) < 30);
}
```

- [ ] **Step 2: Run the regression before changing Blur**

Run: `cmake --build build --target test_editorwindow && QT_QPA_PLATFORM=offscreen ./build/test_editorwindow imageBlurWorksThroughEditorExport`

Expected: PASS on the current tree. If it fails, stop and use its exact pixel output to revise this plan before touching video code.

- [ ] **Step 3: Commit the guard**

```bash
git add tests/test_editorwindow.cpp
git commit -m "test: cover image blur through editor export"
```

### Task 2: Resolve a packaged Tesseract runtime

**Files:**
- Modify: `src/ocr.h`
- Modify: `src/ocr.cpp`
- Modify: `tests/test_ocr.cpp`

**Interfaces:**
- Produces: `OcrRuntime { QString program; QString tessdataDir; }`.
- Produces: `resolveOcrRuntime(const OcrOptions &, const QString &applicationDir, const std::function<bool(const QString &)> &, const std::function<QString(const QString &)> &) -> OcrRuntime`.
- Consumes: existing `OcrOptions::tessdataDir`, `chooseTessdataDir()`, and `QStandardPaths::findExecutable()`.

- [ ] **Step 1: Write resolver tests**

Cover these exact priorities in `tests/test_ocr.cpp`:

```cpp
void resolvesPackagedOcrBeforePath() {
    OcrOptions opts;
    const auto exists = [](const QString &path) {
        return path == QStringLiteral("C:/Eddy/ocr/tesseract.exe")
            || path == QStringLiteral("C:/Eddy/ocr/tessdata/deu.traineddata");
    };
    const auto find = [](const QString &) { return QStringLiteral("C:/PATH/tesseract.exe"); };
    const OcrRuntime runtime = resolveOcrRuntime(opts, QStringLiteral("C:/Eddy"), exists, find);
    QCOMPARE(runtime.program, QStringLiteral("C:/Eddy/ocr/tesseract.exe"));
    QCOMPARE(runtime.tessdataDir, QStringLiteral("C:/Eddy/ocr/tessdata"));
}

void fallsBackToPathOcr() {
    OcrOptions opts;
    const auto never = [](const QString &) { return false; };
    const auto find = [](const QString &) { return QStringLiteral("/usr/bin/tesseract"); };
    const OcrRuntime runtime = resolveOcrRuntime(opts, QStringLiteral("/opt/eddy"), never, find);
    QCOMPARE(runtime.program, QStringLiteral("/usr/bin/tesseract"));
}
```

Also assert that an explicit `tessdataDir` overrides both packaged and system tessdata.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cmake --build build --target test_ocr`

Expected: FAIL to compile because `OcrRuntime` and `resolveOcrRuntime` do not exist.

- [ ] **Step 3: Implement the minimal resolver and use it**

Add the two-field value type and injected resolver in `ocr.h`. In `OcrRunner::recognizeRegion()`, resolve with `QCoreApplication::applicationDirPath()`, `QFileInfo::isFile()`, and `QStandardPaths::findExecutable()`. Start `runtime.program`, and pass `--tessdata-dir runtime.tessdataDir` when non-empty. If `program` is empty, emit `OCR unavailable: tesseract executable not found` without starting a process.

The packaged layout is fixed:

```text
Eddy/
  eddy.exe
  ocr/
    tesseract.exe
    *.dll
    tessdata/
      deu.traineddata
      osd.traineddata
```

- [ ] **Step 4: Run OCR tests**

Run: `cmake --build build --target test_ocr test_redactocrcontroller && QT_QPA_PLATFORM=offscreen ./build/test_ocr && QT_QPA_PLATFORM=offscreen ./build/test_redactocrcontroller`

Expected: PASS.

- [ ] **Step 5: Commit runtime resolution**

```bash
git add src/ocr.h src/ocr.cpp tests/test_ocr.cpp
git commit -m "fix: resolve packaged OCR runtime"
```

### Task 3: Put OCR into both Windows installers

**Files:**
- Modify: `packaging/windows/build-msi.ps1`
- Modify: `packaging/windows/build-nsis.ps1`
- Modify: `.github/workflows/release.yml`
- Modify: `tests/test_redactocrcontroller.cpp`

**Interfaces:**
- Consumes: a release-job directory containing `tesseract.exe`, its runtime DLLs, and `tessdata/deu.traineddata` plus `tessdata/osd.traineddata`.
- Produces: identical `ocr/` directories in MSI and NSIS staging trees.

- [ ] **Step 1: Make both packaging scripts require and stage OCR**

Add a mandatory `TesseractDirectory` parameter. Before calling `windeployqt`, validate these files:

```powershell
$tesseract = Join-Path $TesseractDirectory "tesseract.exe"
$deu = Join-Path $TesseractDirectory "tessdata\deu.traineddata"
$osd = Join-Path $TesseractDirectory "tessdata\osd.traineddata"
foreach ($required in @($tesseract, $deu, $osd)) {
    if (-not (Test-Path -LiteralPath $required -PathType Leaf)) {
        throw "Required OCR file not found: $required"
    }
}
Copy-Item -LiteralPath $TesseractDirectory -Destination (Join-Path $stage "ocr") -Recurse
```

- [ ] **Step 2: Pin and install the release OCR runtime**

In `.github/workflows/release.yml`, download and verify these exact inputs:

```text
https://digi.bib.uni-mannheim.de/tesseract/tesseract-ocr-w64-setup-5.4.0.20240606.exe
sha256 c885fff6998e0608ba4bb8ab51436e1c6775c2bafc2559a19b423e18678b60c9

https://github.com/tesseract-ocr/tessdata_fast/raw/refs/tags/4.1.0/deu.traineddata
sha256 19d219bbb6672c869d20a9636c6816a81eb9a71796cb93ebe0cb1530e2cdb22d
```

Install the NSIS-based Tesseract package to a temporary directory with `/S /D=<absolute-path>`. Create a clean staging directory containing only `tesseract.exe`, its sibling `*.dll` files, `tessdata/osd.traineddata` from the installer, and the pinned German data above; do not ship training executables, documentation, or every language. Pass that staging directory to both packaging scripts. Do not use an unversioned `latest` URL.

- [ ] **Step 3: Add a packaged-runtime smoke test**

After each stage is built and before the installer is created, run:

```powershell
& (Join-Path $stage "ocr\tesseract.exe") --tessdata-dir (Join-Path $stage "ocr\tessdata") --list-langs
if ($LASTEXITCODE -ne 0) { throw "Packaged Tesseract smoke test failed" }
```

Assert the output contains both `deu` and `osd`. Run `test_redactocrcontroller::detectsRealTextEndToEnd` with the staged OCR directory prepended to `PATH`; change that test from `QSKIP` to `QFAIL` when the release job explicitly sets `EDDY_REQUIRE_OCR=1`.

- [ ] **Step 4: Build both installers and inspect their file lists**

Run in the Windows release job: both existing packaging commands with `-TesseractDirectory $env:TESSERACT_RUNTIME`, where the staging step set `TESSERACT_RUNTIME` to its absolute clean runtime directory.

Expected: both artifacts contain `ocr/tesseract.exe`, its DLLs, `ocr/tessdata/deu.traineddata`, and `ocr/tessdata/osd.traineddata`; the OCR smoke test detects rendered `Hallo`.

- [ ] **Step 5: Commit Windows OCR packaging**

```bash
git add packaging/windows/build-msi.ps1 packaging/windows/build-nsis.ps1 .github/workflows/release.yml tests/test_redactocrcontroller.cpp
git commit -m "release: bundle working OCR runtime"
```

### Task 4: Feed current video frames into Redact and OCR

**Files:**
- Modify: `src/items/redactitem.h`
- Modify: `src/items/redactitem.cpp`
- Modify: `src/toolcontroller.h`
- Modify: `src/toolcontroller.cpp`
- Modify: `src/redactocrcontroller.h`
- Modify: `src/redactocrcontroller.cpp`
- Modify: `src/editorwindow.h`
- Modify: `src/editorwindow.cpp`
- Modify: `tests/test_items.cpp`
- Modify: `tests/test_toolcontroller.cpp`
- Modify: `tests/test_editorwindow.cpp`

**Interfaces:**
- Produces: `RedactItem::setSource(const QImage &)` and `RedactItem::blurRectsInScene() const -> QVector<QRect>`.
- Produces: `ToolController::setBackground(const QImage &)` for newly created items.
- Produces: `RedactOcrController::setBackground(const QImage &)` for the next detection.
- Consumes: `QGraphicsVideoItem::videoSink()->videoFrameChanged` and `QVideoFrame::toImage()`.

- [ ] **Step 1: Write source-refresh tests**

In `test_items.cpp`, create a Blur item over a black source, call `setSource()` with a white source, and assert the rendered center changes from dark to light. Assert `blurRectsInScene()` returns the full moved Blur region and only OCR text rectangles for `OcrBlur` after detection.

In `test_toolcontroller.cpp`, call `setBackground(white)` before drawing and assert the new Redact item's blur paints from white, not the constructor's old black image.

- [ ] **Step 2: Run the new tests to verify they fail**

Run: `cmake --build build --target test_items test_toolcontroller`

Expected: FAIL to compile because the new setters/accessor do not exist.

- [ ] **Step 3: Add the three minimal source update methods**

`RedactItem::setSource()` assigns the implicitly shared `QImage` and calls `rebuildCache()` only for Blur modes. `ToolController::setBackground()` and `RedactOcrController::setBackground()` replace their stored `QImage`; they do not create another source-provider class.

`blurRectsInScene()` returns an empty vector for non-blur modes and otherwise maps the existing `coverRects()` results to clipped, aligned scene rectangles. This keeps preview and export on the same coverage rules.

- [ ] **Step 4: Connect the video sink once**

In `EditorWindow::ensureVideoPlayer()`, connect the existing `m_videoItem->videoSink()` before setting the player source:

```cpp
connect(m_videoItem->videoSink(), &QVideoSink::videoFrameChanged,
        this, [this](const QVideoFrame &frame) {
    const QImage image = frame.toImage().convertToFormat(QImage::Format_ARGB32_Premultiplied);
    if (image.isNull()) return;
    m_currentVideoFrame = image;
    m_tools->setBackground(image);
    m_ocr->setBackground(image);
    for (QGraphicsItem *item : m_scene->items())
        if (auto *redact = dynamic_cast<RedactItem *>(item))
            redact->setSource(image);
});
```

Keep the black placeholder only until the first decoded frame. Do not mark a frame refresh as an undoable edit or invalidate the export cache.

- [ ] **Step 5: Verify image and current-frame preview paths**

Run: `cmake --build build --parallel 3 && ctest --test-dir build --output-on-failure -R 'test_(items|toolcontroller|editorwindow|ocr|redactocrcontroller)'`

Expected: PASS. The existing image Blur regression remains green.

- [ ] **Step 6: Commit current-frame support**

```bash
git add src/items/redactitem.h src/items/redactitem.cpp src/toolcontroller.h src/toolcontroller.cpp src/redactocrcontroller.h src/redactocrcontroller.cpp src/editorwindow.h src/editorwindow.cpp tests/test_items.cpp tests/test_toolcontroller.cpp tests/test_editorwindow.cpp
git commit -m "fix: use current video frame for redaction"
```

### Task 5: Export real video blur instead of frozen pixels

**Files:**
- Modify: `src/videoexporter.h`
- Modify: `src/videoexporter.cpp`
- Modify: `src/editorwindow.cpp`
- Modify: `tests/test_videoexporter.cpp`
- Modify: `tests/test_editorwindow.cpp`

**Interfaces:**
- Produces: `VideoExportRequest::blurRects` as `QVector<QRect>` in native video coordinates.
- Consumes: `RedactItem::blurRectsInScene()` from Task 4.
- Preserves: the existing static transparent `overlay`, trim, audio mapping, timeout, and atomic replacement behavior.

- [ ] **Step 1: Write a failing exported-video blur test**

Generate a one-second moving/checker video with ffmpeg, request a center blur rectangle, export it, extract one frame, and compare local contrast inside versus outside the rectangle. The assertion must show lower neighbor variance inside while pixels outside retain the source pattern. Keep the existing static-overlay assertion in the same test run.

Call shape:

```cpp
VideoExportRequest request{input, output, overlay};
request.blurRects = {QRect(16, 12, 32, 24)};
const DeliverResult result = writeVideoWithOverlay(request);
QVERIFY2(result.ok, qPrintable(result.error));
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cmake --build build --target test_videoexporter && QT_QPA_PLATFORM=offscreen ./build/test_videoexporter exportsBlurredRegionsOntoVideo`

Expected: FAIL to compile because `blurRects` does not exist.

- [ ] **Step 3: Add blur rectangles to the existing ffmpeg graph**

For each non-empty rectangle clipped to the native frame, split the current video stream, crop one branch, apply a strong box blur, and overlay that patch back at the same coordinates. Chain multiple regions, then apply the existing PNG overlay last.

The generated graph should have this shape for one region:

```text
[0:v]split=2[base0][crop0];
[crop0]crop=32:24:16:12,boxblur=12:2[blur0];
[base0][blur0]overlay=16:12[v1];
[v1][1:v]overlay=0:0:format=auto:shortest=1[v]
```

Use the bounded ffmpeg expression `min(12\,min(w\,h)/2)` so small rectangles remain valid, and keep `12` as a single named calibration constant beside graph construction. Keep rectangles as numeric arguments built by Eddy; never interpolate file paths into the filter expression.

- [ ] **Step 4: Exclude blur items from the static overlay and pass their geometry**

In `EditorWindow`, gather `blurRectsInScene()` from every `RedactItem` before starting export. Temporarily hide blur-mode items while `renderAnnotationOverlay()` renders the static annotation PNG, then restore their visibility. Populate `VideoExportRequest::blurRects` at the existing `writeVideoWithOverlay()` call site.

Blacken and OCR-Blacken stay in the static overlay. Blur and OCR-Blur are rendered only by ffmpeg, preventing frozen source pixels from covering the video.

- [ ] **Step 5: Run video and editor regressions**

Run: `cmake --build build --parallel 3 && ctest --test-dir build --output-on-failure -R 'test_(videoexporter|editorwindow|items)'`

Expected: PASS, including static overlay, trim/audio, image Blur, current-frame preview, and exported-video blur.

- [ ] **Step 6: Commit frame-aware video export**

```bash
git add src/videoexporter.h src/videoexporter.cpp src/editorwindow.cpp tests/test_videoexporter.cpp tests/test_editorwindow.cpp
git commit -m "fix: export frame-aware video blur"
```

### Task 6: Final release validation and documentation

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: completed packaged OCR and frame-aware video redaction behavior.
- Produces: accurate user-facing requirements and limitations.

- [ ] **Step 1: Document OCR and video semantics**

Add to the Redact documentation:

```markdown
Windows installers include the OCR runtime and German language data. Linux builds
use `tesseract` from `PATH` and require the language selected by `ocr_lang`.

On video, Blur is applied frame-by-frame during export. OCR detects text in the
currently displayed frame and keeps those redaction rectangles fixed for the clip;
it does not track moving text.
```

Also add `ocr_lang` and `ocr_psm` to the existing configuration table.

- [ ] **Step 2: Run the full local suite**

Run: `cmake --build build --parallel 3 && ctest --test-dir build --parallel 3 --output-on-failure`

Expected: 100% pass.

- [ ] **Step 3: Run the Windows release job**

Expected: Windows unit tests pass; required OCR smoke test passes without a skip; MSI and NSIS contain the OCR runtime; both installers are generated.

- [ ] **Step 4: Manual release acceptance**

On a clean Windows VM with no system Tesseract or ffmpeg on `PATH`:

1. Open a screenshot containing German text.
2. Draw Redact and verify Blur visibly obscures the selected image region.
3. Switch to OCR Blur and OCR Blacken; verify only detected text is covered and no `OCR failed` toast appears.
4. Open a video, pause on a text frame, draw Blur, and verify preview follows the displayed frame.
5. Export the video and inspect start/middle/end frames; the selected region stays blurred without frozen pixels.

Expected: all five checks pass. If video cannot open because `ffprobe`/`ffmpeg` is absent, record that as the separate existing Windows video-runtime packaging defect; do not hide it inside this redaction change.

- [ ] **Step 5: Commit documentation**

```bash
git add README.md
git commit -m "docs: describe OCR and video redaction support"
```
