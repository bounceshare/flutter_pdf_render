## Introduction

[pdf_render](https://pub.dartlang.org/packages/pdf_render) is a PDF renderer implementation that supports iOS (>= 8.0) and Android (>= API Level 21). It provides you with intermediate PDF rendering APIs and also easy-to-use Flutter Widgets.

## Widgets

### Note on iOS Simulator

Althouth the plugin uses Flutter's [Texture](https://api.flutter.dev/flutter/widgets/Texture-class.html) to realize fast rendering of PDF pages, it does not work correctly in iOS Simulator and the plugin will fallback to compatibility rendering mode.

Please use the physical device to test the actual behavior.

### Importing Widgets Library

Althouth 0.61.0 introduces new PDF rendering widgets, it also contains deprecated but backward compatible old widgets too. Anyway if you're new to the plugin, you had better use the new widgets with the following import:

```dart
import 'package:pdf_render/pdf_render_widgets2.dart';
```

### Single page view

The following fragment illustrates the easiest way to render only one page of a PDF document using `PdfDocumentLoader`. It is suitable for showing PDF thumbnail.

```dart
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      home: new Scaffold(
        appBar: new AppBar(
          title: const Text('Pdf_render example app'),
        ),
        backgroundColor: Colors.grey,
        body: Center(
          child: PdfDocumentLoader(
            assetName: 'assets/hello.pdf',
            pageNumber: 1,
            pageBuilder: (context, textureBuilder, pageSize) => textureBuilder()
          )
        )
      ),
    );
  }
```

Of course, `PdfDocumentLoader` accepts one of `filePath`, `assetName`, or `data` to load PDF document from a file, or other sources.

### Multipage view using ListView.builder

Using `PdfDocumentLoader` in combination with `PdfPageView`, you can show multiple pages of a PDF document. In the following fragment, `ListView.builder` is utilized to realize scrollable PDF document viewer.

```dart
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      home: new Scaffold(
        appBar: new AppBar(
          title: const Text('Pdf_render example app'),
        ),
        backgroundColor: Colors.grey,
        body: Center(
          child: PdfDocumentLoader(
            assetName: 'assets/hello.pdf',
            documentBuilder: (context, pdfDocument, pageCount) => LayoutBuilder(
              builder: (context, constraints) => ListView.builder(
                itemCount: pageCount,
                itemBuilder: (context, index) => Container(
                  margin: EdgeInsets.all(margin),
                  padding: EdgeInsets.all(padding),
                  color: Colors.black12,
                  child: PdfPageView(
                    pdfDocument: pdfDocument,
                    pageNumber: index + 1,
                  )
                )
              )
            ),
          )
        )
      ),
    );
  }
```

### Customizing page widget

Both `PdfDocumentLoader` and `PdfPageView` accepts `pageBuilder` parameter if you want to customize the visual of each page. The following fragment illustrates that:

```dart
PdfPageView(
  pageNumber: index + 1,
  // pageSize is the PDF page size in pt.
  pageBuilder: (context, textureBuilder, pageSize) {
    //
    // This illustrates how to decorate the page image with other widgets
    //
    return Stack(
      alignment: Alignment.bottomCenter,
      children: <Widget>[
        // the container adds shadow on each page
        Container(
            margin: EdgeInsets.all(margin),
            padding: EdgeInsets.all(padding),
            decoration: BoxDecoration(boxShadow: [
              BoxShadow(
                  color: Colors.black45,
                  blurRadius: 4,
                  offset: Offset(2, 2))
            ]),
            // textureBuilder builds the actual page image
            child: textureBuilder()),
        // adding page number on the bottom of rendered page
        Text('${index + 1}',
            style: TextStyle(fontSize: 50))
      ],
    );
  },
)
```

### textureBuilder

`textureBuilder` (`PdfPageTextureBuilder`) generates the actual widget that directly corresponding to the page image. The actual widget generated may vary upon the situation. But you can of course customize the behavior of the funtion with its parameter.

The function is defined as:

```dart
typedef PdfPageTextureBuilder = Widget Function({Size size, bool returnNullForError, PdfPagePlaceholderBuilder placeholderBuilder});
```

So if you want to generate widget of an exact size, you can specify `size` explicitly.

Please note that the size is in density-independent pixels. The function is responsible for determining the actual pixel size based on device's pixel density.

`returnNullForError` may be true if you want null for PDF page loading/rendering failure; it is, with the parameter, you can handle the behavior on such failures:

```dart
textureBuilder(returnNullForError: true) ?? Container()
```

Ultimately, `placeholderBuilder` is the final resort that controls the "placeholder" for loading or failure cases.

```dart
/// Creates page placeholder that is shown on page loading or even page load failure.
typedef PdfPagePlaceholderBuilder = Widget Function(Size size, PdfPageStatus status);

/// Page loading status.
enum PdfPageStatus {
  /// The page is currently being loaded.
  loading,
  /// The page load failed.
  loadFailed,
}
```

## PDF rendering APIs

The following fragment illustrates overall usage of `PdfDocument`:

```dart
import 'package:pdf_render/pdf_render.dart';

...

/// Open the document using either openFile, openAsset, or openData.
PdfDocument doc = await PdfDocument.openAsset('assets/hello.pdf');

// Get the number of pages in the PDF file
int pageCount = doc.pageCount;

// The first page is 1
PdfPage page = await doc.getPage(1);

// For the render function's return, see explanation below.
PdfPageImage pageImage = await page.render();

// PDFDocument must be disposed as soon as possible.
doc.dispose();
```

And then, you can use `PdfPageImage` to get the actual RGBA image in [dart.ui.Image](https://api.flutter.dev/flutter/dart-ui/Image-class.html).

To embed the image in the widget tree, you can use [RawImage](https://docs.flutter.io/flutter/widgets/RawImage-class.html):

```dart
@override
Widget build(BuildContext context) {
  return Center(
    child: Container(
      padding: EdgeInsets.all(10.0),
      color: Colors.grey,
      child: Center(
        child: RawImage(image: pageImage.image, fit: BoxFit.contain))
    )
  );
}
```

But if you don't want raw RGBA image data access, you had better use faster and efficient `PdfPageImageTexture`.

## PdfDocument.openXXX

On `PdfDocument` class, there are three functions to open PDF from a real file, an asset file, or a memory data.

```dart
// from an asset file
PdfDocument docFromAsset = await PdfDocument.openAsset('assets/hello.pdf');

// from a file
PdfDocument docFromFile = await PdfDocument.openFile('/somewhere/in/real/file/system/file.pdf');

// from PDF memory image on Uint8List
PdfDocument docFromData = await PdfDocument.openData(data);
```

## PdfDocument members

```dart
class PdfDocument {
  final int docId; // For internal purpose
  final int pageCount; // Number of pages in the document
  final int verMajor; // PDF major version
  final int verMinor; // PDF minor version
  final bool isEncrypted; // Whether the file is encrypted or not
  final bool allowsCopying; // Whether the file allows you to copy the texts
  final bool allowsPrinting; // Whether the file allows you to print the document

  // Get a page by page number (page number starts at 1)
  Future<PdfPage> getPage(int pageNumber);

  // Dispose the instance.
  void dispose();
}
```

## PdfPage members

```dart
class PdfPage {
  final int docId; // For internal purpose
  final int pageNumber; // Page number (page number starts at 1)
  final int rotationAngle; // Rotation angle; one of 0, 90, 180, 270
  final double width; // Page width in points; pixel size on 72-dpi
  final double height; // Page height in points; pixel size on 72-dpi

  // render sub-region of the PDF page.
  Future<PdfPageImage> render({
    int x = 0, int y = 0,
    int width = 0, int height = 0,
    double fullWidth = 0.0, double fullHeight = 0.0 });
```

`render` function extracts a sub-region `(x,y)` - `(x + width, y + height)` from scaled `fullWidth` x `fullHeight` PDF page image. All the coordinates are in pixels.

The following fragment renders the page at 300 dpi:

```dart
const scale = 300.0 / 72.0;
const fullWidth = page.width * scale;
const fullHeight = page.height * scale;
var rendered = page.render(
  x: 0,
  y: 0,
  width: fullWidth.toInt(),
  height: fullHeight.toInt(),
  fullWidth: fullWidth,
  fullHeight: fullHeight);
```

## PdfPageImage members

```dart
/// Page number. The first page is 1.
final int pageNumber;
/// Left X coordinate of the rendered area in pixels.
final int x;
/// Top Y coordinate of the rendered area in pixels.
final int y;
/// Width of the rendered area in pixels.
final int width;
/// Height of the rendered area in pixels.
final int height;
/// Full width of the rendered page image in pixels.
final int fullWidth;
/// Full height of the rendered page image in pixels.
final int fullHeight;
/// PDF page width in points (width in pixels at 72 dpi).
final double pageWidth;
/// PDF page height in points (height in pixels at 72 dpi).
final double pageHeight;
/// Rendered image in dart:ui.Image
final Image image;
```

## PdfPageImageTexture members

The class is used to interact with Flutter's [Texture](https://api.flutter.dev/flutter/widgets/Texture-class.html) class to realize faster and resource-saving rendering comparing to PdfPageImage/RawImage combination.

```dart
class PdfPageImageTexture {
  final PdfDocument pdfDocument;
  final int pageNumber;
  final int texId;

  bool operator ==(Object other);
  int get hashCode;

  /// Create a new Flutter [Texture]. The object should be released by calling [dispose] method after use it.
  static Future<PdfPageImageTexture> create({@required PdfDocument pdfDocument, @required int pageNumber});

  /// Release the object.
  Future<void> dispose();

  /// Update texture's sub-rectangle ([destX],[destY],[width],[height]) with the sub-rectangle ([srcX],[srcY],[width],[height]) of the PDF page scaled to [fullWidth] x [fullHeight] size.
  /// If [backgroundFill] is true, the sub-rectangle is filled with white before rendering the page content.
  /// The method can also resize the texture if you specify [texWidth] and [texHeight].
  Future<void> updateRect({int destX = 0, int destY = 0, int width, int height, int srcX = 0, int srcY = 0, int texWidth, int texHeight, double fullWidth, double fullHeight, bool backgroundFill = true});
}
```

## Future plans

- Supporting password protected PDF files (#1)
