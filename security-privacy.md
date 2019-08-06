Questions from https://www.w3.org/TR/security-privacy-questionnaire/

# 2. Questions to Consider

## 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This feature intentionally reveals a list of local fonts to the Web site. This can include common fonts, fonts purchased from type foundries, or even custom fonts such as personal handwriting fonts.

Browsers currently provide for the use of local fonts, but not enumeration. For example, the CSS `font-family` property can be used to request the use of a local font by name. If the font is not available, the browser will provide a fallback. Through the use of measurement APIs, it is usually possible to determine if the requested font or a fallback was used. Given a dictionary of font names, this can be used to determine which are available on a user's system.

The feature will require the user to grant permission before providing the data to a site.

## 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

The feature exposes the names and a handful of additional properties of each font. For example, the name, "PostScript" name, metrics, color and variability information. These are needed by Web applications that will present a list of fonts to users - e.g. illustration tools - to group and classify options. Some of these properties would also be available indirectly e.g. through measurement APIs.

## 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

There are services which create fonts based on handwriting samples. If these fonts are given names including personally identifiable information (e.g. "Alice's Handwriting Font"), then personally identifiable information would be made available. This may not be apparent to users if the information is included as properties within the font, not just the font name.

User agents should make the risks of granting the permission clear to users.

## 2.4. How does this specification deal with sensitive information?

Fonts installed on particular operating system versions could reveal information about the user's location.

Fonts may be installed by particular applications installed on the system, for example office suites. This could allow identifying the other applications on the system.

Users from a particular organization could have specific fonts installed. Employees of "Example Co." could all have an "Example Corporate Typeface" installed by their system administrator, which would allow distinguishing users of a site as employees.

User agents should make the risks of granting the permission clear to users.

## 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

Yes, user agents could persist the `local-fonts` permission grant, but at least in the Chrome implementation, this permission grant will only be persistent for installed PWAs. The drive-by web will only have enough state to allow it to re-prompt for access, but the access itself won't be persistent.

Furthermore, the user will be able to revoke permission to clear the state that was persisted, similarly to how other permissions work.

## 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

The font list includes:

* Fonts included in the operating system distribution.
* Fonts installed by particular applications installed on the system, for example office suites.
* Fonts directly installed by the system administrator and/or end user.

This will identify the operating system and version and potentially some installed applications.

## 2.7. Does this specification allow an origin access to sensors on a user’s device

No.

## 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

The data described above is exposed to script, which can then transmit it to the origin.

## 2.9. Does this specification enable new script execution/loading mechanisms?

No.

## 2.10. Does this specification allow an origin to access other devices?

No.

## 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No. (Other than potentially triggering the display of a permission prompt.)

## 2.12. What temporary identifiers might this this specification create or expose to the web?

None.

## 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

A user agent may decline to grant permissions requested by third-party contexts. Cooperating origins could work around this limitation via `postMessage()`.

## 2.14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?

The user agent could automatically deny the permission request. The user agent could also grant the permission request, but provide "anonymous" data, e.g. a fixed set of fonts, rather than enumerating the actual local fonts.

## 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

TBD.

## 2.16. Does this specification allow downgrading default security characteristics?

No.

## 2.17. What should this questionnaire have asked?



# 3. Threat Models

## 3.1 Passive Network Attackers

## 3.2 Active Network Attackers

## 3.3 Same-Origin Policy Violations

## 3.4 Third-Party Tracking

## 3.5 Legitimate Misuse
