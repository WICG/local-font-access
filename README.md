<img src="https://inexorabletash.github.io/font-enumeration/logo-font-enumeration.svg" height=100 align=right>

# Local Font Access Explained

> August 14th, 2018<br>
> Last Update: March 23rd, 2020
>
> Alex Russell `<slightlyoff@google.com>`<br>
> Emil A Eklund `<eae@google.com>`<br>
> Josh Bell `<jsbell@google.com>`<br>
> Chase Phillips `<cmp@google.com>`<br>
> Olivier Yiptong `<oyiptong@google.com>`<br>

## What’s all this then?

Professional-quality design and graphics tools have historically been difficult to deliver on the web.

One stumbling block has been an inability to access and use the full variety of professionally constructed and hinted fonts which designers have locally installed. The web's answer to this situation has been the introduction of [Web Fonts](https://developer.mozilla.org/en-US/docs/Learn/CSS/Styling_text/Web_fonts) which are loaded dynamically by browsers and are subsequently available to use via CSS. This level of flexibility enables some publishing use-cases but fails to fully enable high-fidelity, platform independent vector-based design tools for several reasons:

 * System font engines (and browser stacks) may handle the parsing and display of certain glyphs differently. These differences are necessary, in general, to create fidelity with the underlying OS (so web content doesn't "look wrong"). These differences reduce fidelity.
 * Developers may have legacy font stacks for their applications which they are bringing to the web. To use these engines, they usually require direct access to font data; something Web Fonts do not provide.

We propose a two-part API to help address this gap:

 * A font enumeration API which may, optionally, allow users to grant access to the full set of available system fonts in addition to network fonts.
 * From each enumeration result, the ability to request low-level (byte-oriented) access to the various [TrueType/OpenType](https://docs.microsoft.com/en-us/typography/opentype/spec/otff#font-tables) tables.

Taken together, these provide high-end tools access to the same underlying data tables that browser layout and rasterization engines use for drawing text. Examples of these data tables include the [glyf](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf) table for glyph vector data, the GPOS table for glyph placement, and the GSUB table for ligatures and other glyph substitution. This information is necessary for these tools in order to guarantee both platform-independence of the resulting output (by embedding vector descriptions rather than codepoints) and to enable font-based art (treating fonts as the basis for manipulated shapes).

Note that this implies that the web application provides its own shaper and libraries for Unicode, bidirectional text, text segmentation, and so on, duplicating the user agent and/or operating system's text stack. See the "Considered alternatives" section below.

> NOTE: Long term, we expect that this proposal would merge into an existing CSS-related spec rather than stand on its own.

### Goals

A successful API should:

 * Where allowed, provide efficient enumeration of all local fonts without blocking the main thread
 * Ensure UAs are free to return anything they like. If a browser implementation prefers, they may choose to only provide a set of default fonts built into the browser.
 * Be available from Workers
 * Allow multiple levels of privacy preservation; e.g. full access for "trusted" sites and degraded access for untrusted scenarios
 * Reflect local font access state in the [Permissions API](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API)
 * Provide the ability to uniquely identify a specific font in the case of conflicting names (e.g. Web Font aliases vs. local PostScript font names)
 * Enable access to all [browser-allowed font tables](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) (may vary per browser)
 * Enable a memory efficient implementation, avoiding leaks and copies by design
 * Shield applications from unnecessary complexity by requiring that browser implementations produce valid OpenType data in the returned tables
 * Restrict access to local font data to Secure Contexts and to only the top-most frame by default via the [Feature Policy](https://wicg.github.io/feature-policy) spec
 * Sort any result list by font name to reduce possible fingerprinting entropy bits; e.g. .query() returns an iterator which will be sorted by Unicode code point given font names


#### Possible/Future Goals

 * Direct access to localized font names (can be done via table API)
 * Access to font tables for web (network-loaded) fonts
 * Registration of new font families (extensibility)
 * Additional metadata available during enumeration (ascender, descender, baseline, x-height, etc.). Will require feedback from developers; can be determined using via the [font-table-access API](https://github.com/inexorabletash/font-table-access) even if not exposed during enumeration.
 * Signals when system font configuration changes (fonts added/removed); some designers work with tools that swap font portfolios at the system level
 * Provide access to named instances and subfamilies (e.g. "semibold", "light")

### Non-goals

This API will not try to:

 * Fully describe how font loading works within the web platform. Fonts are a complex topic and Web Font loading implicates aspects of layout and style recalculation which are not at this time pluggable. As this design isn't addressing those aspects, we will not describe font application or CSS recalculation semantics
 * Standardize font family detection or grouping
 * Describe or provide full access to an existing WOFF/TTF/PS parser.
 * Provide access to the underlying WOFF/TTF/PS font files or describe their locations on disk.
 * Provide a guarantee that the set of available tables or their content matches the font on disk byte to byte.
 * Normalize differences in processed font tables across browser implementations. The font tables that will be exposed will have been processed by browser-provided parsers, but we will not describe or constrain them except to say that their output will continue to be in a valid OpenType format. For instance, if a library like [OTS](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) reduces the available information for a font, this spec will not require implementations to do more than they already would or provide alternative ways of getting such information back from the source font files.

## Key scenarios

> Note: Earlier versions of this document attempted to sketch out two versions of each API; one based on `FontFaceSource` and the other the fully-asynchronous version that survives in this doc. While attractive from a re-use perspective, [`FontFaceSource`](https://drafts.csswg.org/css-font-loading/#font-face-source) (and the implied global `window.fonts`) implies synchronous iteration over a potentially unbounded (and perhaps slow) set of files, and each item may require synchronous IPCs and I/O. This, combined with the lack of implementations of `FontFaceSource` caused us to abandon this approach.

### Enumerating Local Fonts

Web developers historically lack anything more than heuristic information about which local fonts are available for use in styling page content. Web developers often include complex lists of `font-family` values in their CSS to control font fallback in a heuristic way. Generating good fallbacks is such a complex task for designers that [tools have been built to help "eyeball"](https://meowni.ca/font-style-matcher/) likely-available local matches.

Font enumeration can help by enabling:

 * Logging of likely-available fonts to improve server-side font rule generation.
 * Scripts to generate style rules based on "similar" local fonts, perhaps saving a download
 * Improving styling options for user-generated content, allowing the generation of style rules via more expressive font selection menus

```js
// Asynchronous Query and Iteration
(async () => { // Async block
  // May prompt the user:
  const status = await navigator.permissions.request({ name: "local-fonts" });
  if (status.state !== "granted")
    throw new Error("Cannot enumerate local fonts");

  // This sketch returns individual FontFace instances rather than families:
  // In the future, query() could take filters e.g. family name, and/or options
  // e.g. locale.
  const fonts_iterator = navigator.fonts.query();

  for await (const metadata of fonts_iterator) {
    console.log(metadata.postscriptName);
    console.log(metadata.fullName);
    console.log(metadata.family);
  }
})();
```

### Styling with Local Fonts

Advanced creative tools may wish to use CSS to style text using all available local fonts. In this case, getting access to the local font name can allow the user to select from a richer set of choices:

```js
const font_select = document.createElement("select");
font_select.onchange = e => {
  console.log("selected:", font_select.value);
  // Use the selected font to style something here.
};

document.body.appendChild(font_select);

(async () => { // Async block
  // May prompt the user:
  const status = await navigator.permissions.request({ name: "local-fonts" });
  if (status.state !== "granted")
    throw new Error("Cannot continue to style with local fonts");

  for await (const metadata of navigator.fonts.query()) {
    const option = document.createElement("option");
    option.text = metadata.fullName;
    option.value = metadata.fullName;
    option.setAttribute("postscriptName", metadata.postscriptName);
    font_select.append(option);
  }
})();
```

### Accessing Font Tables

Here we use enumeration and new APIs on `FontFace` to access specific OpenType tables of local fonts; we can use this to parse out specific data or feed it into, e.g., WASM version of [HarfBuzz](https://www.freedesktop.org/wiki/Software/HarfBuzz/) or [Freetype](https://www.freetype.org/):

```js
(async () => { // Async block
  // May prompt the user
  const status = await navigator.permissions.request({ name: "local-fonts" });
  if (status.state !== "granted")
    throw new Error("Cannot continue to style with local fonts");

  for await (const metadata of navigator.fonts.query()) {
    // Looking for a specific font:
    if (metadata.postscriptName !== "Consolas")
      continue;

    // 'getTables()' returns Blobs of table data. The default is
    // to return all available tables. See:
    //    https://docs.microsoft.com/en-us/typography/opentype/spec/
    // Here we ask for a subset of the tables:
    const tables = await metadata.getTables([ "glyf", "cmap", "head" ]);

    // 'tables' is a Map of table names to Blobs
    const blob = tables.get("head");

    // Slice out only the bytes we need.
    const bytes = new DataView(await blob.slice(0, 4).arrayBuffer());

    // Parse out the version number of our font:
    //    https://docs.microsoft.com/en-us/typography/opentype/spec/head
    const major = bytes.getInt16(0);
    const minor = bytes.getInt16(2);
    console.log("Consolas version:", (major + (minor/10)));
  }
})();
```

## Detailed design discussion (tables)

Several aspects of this design need validation:

* The arguments to `getTables()` could easily be done a dozen different ways. Feedback appreciated.
* This design tries to address concerns with `FontFaceSet` and friends at the cost of introducing a new API surface.
* It isn't strictly clear that providing table-by-table access is the best choice.
* The specifics of table filtering (i.e. what tables/subtables are included) is left up to user agents.


Other issues that feedback is needed on:

* Enumeration order of the returned table map needs to be defined.


## Detailed design discussion (enumeration)

Several aspects of this design need validation:

* What precisely is being iterated over needs to be identified. Is it over files on disk, families, or other groupings that a system level enumeration API provides? There is not a 1:1 relationship between files and named instances.
* Grouping of related fonts and variants into a parent object is difficult. Some families can be represented by one file or many, and the definition of a "family" is heuristic to start with. Is grouping needed? Design currently leaves this open to future additions.
* `FontFace` objects provide a lot of metadata synchronously, by default. While this sketch provides additional metadata asynchronously, is using `FontFace` with the sync data subset a problem?
* This design tries to address concerns with `FontFaceSet` and friends at the cost of introducing a new API surface.

Other issues that feedback is needed on:

* Font "name" propertes in OpenType are quite logically a map of (language tag → string) rather than just a string. The sketch just provides a single name (the "en" variant or first?) - should we introduce a map? Or have `query()` take a language tag? Or defer for now?


### Privacy and Security Considerations

* The `local-fonts` permission appears to provide a highly fingerprintable surface. However, UAs are free to return anything they like.  For example, the Tor Browser or Brave may choose to only provide a set of default fonts built into the browser. Similarly, UAs are not required to provide table data exactly as it appears on disk. Chrome, e.g., will only provide access to table data after sanitization via [OTS](https://github.com/khaledhosny/ots) and will fail to reflect certain tables entirely.

* Some users (mostly in big organizations) have custom fonts installed on their system.  Listing these could provide highly identifying information about the user's company.

* Wherever possible, these APIs are designed to only expose exactly the information needed to enable the mentioned use cases.  System APIs may produce a list of installed fonts not in a random or a sorted order, but in the order of font installation.  Returning exactly the list of installed fonts given by such a system API can expose additional entropy bits, and use cases we want to enable aren't assisted by retaining this ordering.  As a result, this API requires that the returned data be sorted before being returned.

## Considered alternatives

### `FontFaceSource`

[`FontFaceSource`](https://drafts.csswg.org/css-font-loading/#font-face-source) is specified in the [CSS 3 Font Loading draft](https://drafts.csswg.org/css-font-loading/). At first glance, this is the most appropriate interface from which to hang something like the proposed `query()` method. It is, however, a synchronous iterator. In conversation with implemeners, this contract may be problematic from a performance perspective across OSes. Instead of providing a potentially unbounded way for developers to naively lock up the main thread, we've chosen to introduce a different root object from which to hang asynchronous iteratation and query methods.

This might be the wrong thing to do! Hopefully vendors can weigh in more thoroughly on this point.

### Add a browser/OS-provided font chooser

The proposed API exposes some more bits about the user via the web that could
improve fingerprinting efforts.  The bits are based on the presence or lack of
presence of certain fonts in the enumeration-returned list.

An alternative to the API that only exposes a single user-selected font was
considered.  This alternative enumeration API would trigger a
browser/OS-provided font chooser and, from that chooser, the user would select
a single font.  This would reduce the bits exposed to help mitigate
fingerprinting at the cost of significant new functionality.

We've heard interest from partners in a full-fledged enumeration API to get
access to the list of available fonts on the system, and haven't heard interest
in a font-chooser approach to the enumeration API.  However, we're keeping the
alternative in mind as we balance the need for new functionality with privacy
concerns.

### Metadata Properties

Including a subset of useful font metrics (`ascender`, `descender`, `xheight`, `baseline`) in the metadata was considered. Some are complicated (`baseline`), others more straightforward but may not be of practical use, especially if the full pipeline involves passing tables into Harfbuzz/FreeType for rendering. They are not included in the latest version of the sketch.

Additional metadata properties such whether the font uses color (`SBIX`, `CBDT`, `SVG` etc), or is a variable font could be provided, but may not be of use.

## Exposing Building Blocks

To be of use, font table data must be consumed by a shaping engine such as [HarfBuzz](https://www.freedesktop.org/wiki/Software/HarfBuzz/), in conjunction with Unicode libraries such as [ICU](http://site.icu-project.org/home) for bidirectional text support, text segmentation, and so on. Web applications could include these existing libraries, for example compiled via WASM, or equivalents. Necessarily, user agents and operating systems already provide this functionality, so requiring web applications to include their own copies leads to additional download and memory cost. In some cases, this may be required by the web application to ensure identical behavior across browsers, but in other cases exposing some of these libraries directly to script as additional web APIs could be beneficial.

(Parts of ICU are being incrementally exposed to the web via the [ECMA-402](https://ecma-international.org/ecma-402/) effort.)

## References & acknowledgements

The following references have been invaluable:

* [MSDN DirectWrite overview](https://docs.microsoft.com/en-us/windows/desktop/directwrite/introducing-directwrite#accessing-the-font-system)
* [OpenType Specification](https://docs.microsoft.com/en-us/typography/opentype/spec/)
* [OpenType Font Table overview](https://docs.microsoft.com/en-us/typography/opentype/spec/chapter2)

We'd like to acknowledge the contributions of:

* Daniel Nishi, Owen Campbell-Moore, and Mike Tsao who helped pioneer the previous local font access proposal
* Evan Wallace, Biru, Leah Cassidy, Katie Gregorio, Morgan Kennedy, and Noah Levin of Figma who have patiently enumerated the needs of their ambitious web product.
* Tab Atkins and the CSS Working Group who have provided usable base-classes which only need slight extension to enable these cases
* Dominik Röttsches and Igor Kopylov for their thoughtful feedback
