# Web Font Listing and Font Table Access Explained

> August 14th, 2018<br>
> Last Update: March 5th, 2019
>
> Alex Russell <code>&lt;slightlyoff@google.com&gt;</code><br>
> Emil A Eklund <code>&lt;eae@google.com&gt;</code><br>
> Josh Bell <code>&lt;jsbell@google.com&gt;</code><br>

## What’s all this then?

Professional-quality design and graphics tools have historically been difficult to deliver on the web.

One stumbling block has been an inability to access and use the full variety of professionally constructed and hinted fonts which designers have locally installed. The web's answer to this situation has been the introduction of [Web Fonts](https://developer.mozilla.org/en-US/docs/Learn/CSS/Styling_text/Web_fonts) which are loaded dynamically by browsers and are subsequently available to use via CSS. This level of flexibility enables some publishing use-cases but fails to fully enable high-fidelity, platform independent vector-based design tools for several reasons:

 * System font engines (and browser stacks) may handle the parsing and display of certain glyphs differently. These differences are necessary, in general, to create fidelity with the underlying OS (so web content doesn't "look wrong"). These differences reduce fidelity.
 * Developers may have legacy font stacks for their applications which they are bringing to the web. To use these engines, they usually require direct access to font data; something Web Fonts do not provide.

We propose two cooperating APIs to help address this gap:

 * A font-enumeration API which may, optionally, allow users to grant access to the full set of available system fonts in addition to network fonts
 * A font-table-access API which provides low-level (byte-oriented) access to the various [TrueType/OpenType](https://docs.microsoft.com/en-us/typography/opentype/spec/otff#font-tables) tables of both local and remotely-loaded fonts

Taken together, these APIs provide high-end tools access to the same underlying data tables that browser layout and rasterization engines use for drawing text. Such as the [glyf](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf) table for glyph vector data, the GPOS table for glyph placement, and the GSUB table for ligatures and other glyph substitution. This information is necessary for these tools in order to guarantee both platform-independence of the resulting output (by embedding vector descriptions rather than codepoints) and to enable font-based art (treating fonts as the basis for manipulated shapes).

### Goals

A successful API should enable:

 * An efficient enumeration of all local fonts without blocking the main thread
 * Access to named instances and subfamilies (e.g. "semibold", "light")
 * Multiple levels of privacy preservation; e.g., full access for "trusted" sites and degraded access for untrusted scenarios
 * Access to all [browser-allowed font tables](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) (may vary per browser)
 * The ability to uniquely identify a specific font in the case of conflicting names (e.g., Web Font aliases vs. local PostScript font names)
 * Unique identification of families and instances (variants like "bold" and "italic"), including PostScript names
 * Easy identification of [variable](https://developers.google.com/web/fundamentals/design-and-ux/typography/variable-fonts/) and colour ([COLR](https://docs.microsoft.com/en-us/typography/opentype/spec/colr), [CBDT](https://docs.microsoft.com/en-us/typography/opentype/spec/cbdt), [sbix](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6sbix.html)) fonts
 * Re-use of Web Font types and interfaces to the greatest extent possible
 * Reflect local font access state in the [Permissions API](https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API)
 * Restrict access to local font data to Secure Contexts
 * Availability from Workers

#### Possible Goals

 * Registration of new font families via the Tables API (extensibility)
 * Rich metadata available during enumeration (ascender, descender, baseline, x-height, etc.). Will require feedback from developers.


### Non-goals

These APIs will not try to:

 * Fully describe how font loading works within the web platform. Fonts are a complex topic and Web Font loading implicates aspects of layout and style recalculation which are not at this time pluggable. As this design isn't addressing those aspects, we will not describe font application or CSS recalculation semantics
 * Describe or provide full access to an existing WOFF/TTF/PS parser
 * Normalize differences in resulting font tables across implementations (see previous point). The resulting font families, variations, and tables that will be exposed will have been processed by browser-provided parsers, but we will not describe or constrain them. For instance, if a library like [OTS](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) reduces the available information for a font, this spec will not require implementations to do more than they already would or provide alternative ways of getting such information back from the source font files.
 * Provide raw access to the raw bytes of underlying WOFF/TTF/PS font files or describe their locations on disk
 * Standardize font family detection or grouping

## Key scenarios

> Note: Earlier versions of this document attempted to sketch out two versions of each API; one based on `FontFaceSource` and the other the fully-asynchronous verison that survives in this doc. While attractive from a re-use perspective, [`FontFaceSource`](https://drafts.csswg.org/css-font-loading/#font-face-source) (and the implied global `window.fonts`) implies synchronous iteration over a potentially unbounded (and perhaps slow) set of files, and each item may require synchronous IPCs and I/O. This, combined with the lack of implementations of `FontFaceSource` caused us to abandon this approach.

### Styling With Local Fonts

Web developers historically lack anything more than heuristic information about which local fonts are available for use in styling page content. Web developers often include complex lists of `font-family` values in their CSS to control font fallback in a heuristic way. Generating good fallbacks is such a complex task for designers that [tools have been built to help "eyeball"](https://meowni.ca/font-style-matcher/) likely-available local matches.

Font enumeration can help by enabling:

 * Logging of likely-available fonts to improve server-side font rule generation.
 * Scripts to generate style rules based on "similar" local fonts, perhaps saving a download
 * Improving styling options for user-generated content, allowing the generation of style rules via more expressive font selection menus

```js
// Asynchronous Query and Iteration
(async () => { // Async block
  // This sketch returns individual FontFace instances rather than families:
  let fontsIterator = navigator.fonts.query({
                        family: "*",
                        /* example query params; names inspired by CSS:
                        style: [ "italic" ],
                        weight: [ 100, 400, 900, "bold" ],
                        stretch: [ "condensed", "normal", "expanded" ],
                        // TODO: Missing query params?
                        */
                      });

  // Async Iterator syntax is optional as the return value is a generator
  // that yields values that conform to the Promise-based `{ value, done }`
  // protocol:
  //    https://github.com/tc39/proposal-async-iteration
  for await (let f of fontsIterator) {
    f.getMetaData().then((m) => {
      console.log(f.family);         // The given "family" name
      // NEW metadata:
      console.log(m.instance);
      console.log(m.postScriptName);
      console.log(m.localizedName);
      console.log(m.ascender);  // TODO: define units and values
      console.log(m.descender); // TODO: define units and values
      console.log(m.baseline);  // TODO: define units and values
      console.log(m.xheight);   // TODO: define units and values
      console.log(m.isVariable);// TODO: boolean enough?
      console.log(m.isColor);   // TODO: boolean enough?
      // ...
    });
  }
})();
```

### Styling with Local Fonts

Advanced creative tools may wish to use CSS to style text using all available local fonts. In this case, getting access to the local font name can allow the user to select from a richer set of choices:

```js
let fontContainer = document.createElement("select");
fontContainer.onchange = (e) => {
  console.log("selected:", fontContainer.value);
  // Use the selected font to style something here.

  document.body.appendChild(fontContainer);

  let baseFontOption = document.createElement("option");

  (async () => { // Async block
    // May prompt the user:
    let status = await navigator.permissions.request({ name: "local-fonts" });
    if (status.state != "granted") {
      throw new Error("Cannot continue to style with local fonts");
    }
    // TODO(slightlyoff): is this expressive enough?
    for await (let f of navigator.fonts.query({
                          family: "*",
                          local: true,
                        })) {
      f.getMetaData().then((metadata) => {
        console.log(f.family);
        console.log(metadata.instance);
        console.log(metadata.postScriptName);

        option.text = f.family;
        option.value = f.family;
        option.setAttribute("postScriptName", f.postScriptName);
      });
    }
  })();
};
```

### Accessing Font Tables

Here we use enumeration and new APIs on `FontFace` to access specific OpenType tables of local fonts; we can use this to parse out specific data or feed it into, e.g., WASM version of [HarfBuzz](https://www.freedesktop.org/wiki/Software/HarfBuzz/) or [Freetype](https://www.freetype.org/):

```js
(async () => { // Async block
  // May prompt the user
  let status = await navigator.permissions.request({ name: "local-fonts" });
  if (status.state != "granted") {
    throw new Error("Cannot continue to style with local fonts");
  }
  for await (let f of navigator.fonts.query({
                        family: "Consolas.*",
                        local: true,
                      })) {
    // `getTables` returns ArrayBuffers of table data. The default is
    // to return all available tables. See:
    //    https://docs.microsoft.com/en-us/typography/opentype/spec/
    // Here we ask for a subset of the tables:
    f.getTables(["glyf", "cmap", "head"], {/*options*/}).then((tables) => {
      // `tables` is a Map of table names to ArrayBuffers
      let head = new DataView(tables.get("head"));
      // Parse out the version number of our font:
      //    https://docs.microsoft.com/en-us/typography/opentype/spec/head
      let major = head.getInt16(0);
      let minor = head.getInt16(2);
      console.log("Consolas version:", (major + (minor/10)));
    });
  }
})();
```

## Detailed design discussion

Several aspects of this design need validation:

  - Grouping of related fonts and variants into a parent object is difficult. Some families can be represented by one file or many, and the definition of a "family" is heuristic to start with. Is grouping needed? Design currently leaves this open to future additions.
  - `FontFace` objects provide a lot of metadata synchronously, by default. Is this a problem?
  - Many aspects of the `navigator.fonts.query()` method signature are shaky. Are these the right options?
  - Similarly, the arguments to `getTables()` could easily be done a dozen different ways. Feedback appreciated.
  - This design tries to address concerns with `FontFaceSet` and friends at the cost of introducing a new API surface.
  - It isn't strictly clear that providing table-by-table access is the best choice.

### Privacy and Security Considerations

The `local-fonts` permission appears to provide a highly fingerprintable surface. However, UAs are free to return anything they like.

For example, the Tor Browser or Brave may choose to only provide a set of default fonts built into the browser. Similarly, UAs are not required to provide table data exactly as it appears on disk. Chrome, e.g., will only provide access to table data after sanitization via [OTS](https://github.com/khaledhosny/ots) and will fail to reflect certain tables entirely.

## Considered alternatives

### `FontFaceSource`

[`FontFaceSource`](https://drafts.csswg.org/css-font-loading/#font-face-source) is specified in the [CSS 3 Font Loading draft](https://drafts.csswg.org/css-font-loading/). At first glance, this is the most appropriate interface from which to hang something like the proposed `query()` method. It is, however, a synchronous iterator. In conversation with implemeners, this contract may be problematic from a performance perspective across OSes. Instead of providing a potentially unbounded way for developers to naively lock up the main thread, we've chosen to introduce a different root object from which to hang asynchronous iteratation and query methods.

This might be the wrong thing to do! Hopefully vendors can weigh in more thoroughly on this point.

### Raw Font File Access

A previous design for [Local Font Access](https://github.com/DHNishi/LocalFontAccess/blob/master/explainer.md) was designed to explicitly provide access to the underlying bytes of the font file without parsing or metadata extraction. This is arguably a better layering, in that it's possible to build all of the proposed methods for extracting metadata on top of such a system, however it received [significant push-back from experts in the domain](https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/G-hC66MRTso/uVrmHV0NAwAJ).

### Glyph Vector Data Access

Processing of font table data eventually ends up with a vectorized description of the glyph to be painted. This vectorized representation is the goal of professional design software in getting access to low-level font descriptions, however the engines that turn font tables into glyphs do not do so consistently across browsers and OSes.

Professional design packages therefore contain their own table-to-glyph conversion code (frequently by embedding a copy of [FreeType](https://www.freetype.org/)) to guarantee consistency. While it may be a valuable next step to also deliver this higher-level information for developers who want it, we have not had requests yet for such an API, whereas we have engaged and excited users asking for font table access.

### FreeType/HarfBuzz APIs (Roughly)

Chromium embeds FreeType and Harfbuzz to handle font parsing, font metrics computation, line-breaking, HTML/CSS and OpenType layout, text shaping and many other font-related tasks. Why not simply expose APIs that look very similar to theirs? A few reasons. First, these low-level systems may not be what other vendors implement, making their standardisation premature or inappropriate (depending). Secondly, these interfaces may change. Better in that case to design the abstractions we need, allowing our implementations to change without undue stress. Lastly, the calling conventions of these APIs is not compatible with JavaScript's cooperative multi-tasking, meaning we'd need to re-design wrappers for them anyhow.

## References & acknowledgements

The following references have been invaluable:

  - [MSDN DirectWrite overview](https://docs.microsoft.com/en-us/windows/desktop/directwrite/introducing-directwrite#accessing-the-font-system)
  - [OpenType Specification](https://docs.microsoft.com/en-us/typography/opentype/spec/)
  - [OpenType Font Table overview](https://docs.microsoft.com/en-us/typography/opentype/spec/chapter2)

We'd like to acknowledge the contributions of:

  - Daniel Nishi, Owen Campbell-Moore, and Mike Tsao who helped pioneer the previous local font access proposal
  - Evan Wallace, Biru, Leah Cassidy, Katie Gregorio, Morgan Kennedy, and Noah Levin of Figma who have patiently enumerated the needs of their ambitious web product.
  - Tab Atkins and the CSS Working Group who have provided usable base-classes which only need slight extension to enable these cases
  - Dominik Röttsches and Igor Kopylov for their thoughtful feedback
