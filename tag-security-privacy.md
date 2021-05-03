# TAG Security & Privacy Questionnaire Answers #

* **Author:** wanderview@chromium.org
* **Questionnaire Source:** https://www.w3.org/TR/security-privacy-questionnaire/#questions

## Questions ##

* **What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**
  * This feature introduces partitioning for a number of APIs restricting information available across sites.  We do not believe it exposes new information.
* **Is this specification exposing the minimum amount of information necessary to power the feature?**
  * We believe so.  We could have attempted to make partitions based on the top-level origin instead of top-level site, but our research indicated this would have been too breaking for many existing sites.  In addition, other browsers have largely settled on partitioning by top-level site.
* **How does this specification deal with personal information or personally-identifiable information or information derived thereof?**
  * This feature does not collect any personally-identifiable information.
* **How does this specification deal with sensitive information?**
  * This feature does not directly deal with sensitive information.  Partitioning, however, does make it harder to communicate sensitive information across site boundaries.
* **Does this specification introduce new state for an origin that persists across browsing sessions?**
  * This feature does create new partitions for origins loaded in 3rd party contexts that persist across browsing sessions.
* **What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?**
  * While this feature does not directly expose any information about the underlying platform, we do allow extensions to load origins they have host permission access to in a first party context.  This could leak the existence of the extension to the site.  The extension, of course, could use its permission in other ways to result in the same thing by injecting code, etc.
* **Does this specification allow an origin access to sensors on a user’s device**
  * No.
* **What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**
  * This information maintains all current same-origin restrictions on data access and adds additional partitioning based on top-level site for 3rd party contexts.  No need information is exposed to the origin.
* **Does this specification enable new script execution/loading mechanisms?**
  * No.
* **Does this specification allow an origin to access other devices?**
  * No.
* **Does this specification allow an origin some measure of control over a user agent’s native UI?**
  * No.
* **What temporary identifiers might this this specification create or expose to the web?**
  * The partitions are not labeled or directly observable.  Until 3rd party cookies are fully deprecated it will be possible for sites to detect partitioning, but eventually we expect other state to be restricted such that partitions cannot be observed.
* **How does this specification distinguish between behavior in first-party and third-party contexts?**
  * The main goal of the feature is to partition APIs and state in 3rd party contexts.
* **How does this specification work in the context of a user agent’s Private \ Browsing or "incognito" mode?**
  * Partitioning should work the same way in incognito mode.  State is partitioned, but also becomes ephemeral.
* **Does this specification have a "Security Considerations" and "Privacy Considerations" section?**
  * Yes.
* **Does this specification allow downgrading default security characteristics?**
  * No.
