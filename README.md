# Explainer for the Web Translation API

_This proposal is an early design sketch by the Chrome translate API team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome._

Browsers are increasingly offering language translation to their users. Such translation capabilities can also be useful to web developers. This is especially the case when browser's built-in translation abilities cannot help, such as:

* translating user input or other interactive features;
* pages with complicated DOMs which trip up browser translation;
* providing in-page UI to start the translation; or
* translating content that is not in the DOM, e.g. spoken content.

To perform translation in such cases, web sites currently have to either call out to cloud APIs, or bring their own translation models and run them using technologies like WebAssembly and WebGPU. This proposal introduces a new JavaScript API for exposing a browser's existing language translation abilities to web pages, so that if present, they can serve as a simpler and less resource-intensive alternative.

## Goals

Our goals are to:

* Help web developers perform real-time translations (e.g. of user input).
* Help web developers perform real-time language detection.
* Guide web developers to gracefully handle failure cases, e.g. translation not being available or possible.
* Harmonize well with existing browser and OS translation technology ([Brave](https://support.brave.com/hc/en-us/articles/8963107404813-How-do-I-use-Brave-Translate), [Chrome](https://support.google.com/chrome/answer/173424?hl=en&co=GENIE.Platform%3DDesktop#zippy=%2Ctranslate-selected-text), [Edge](https://support.microsoft.com/en-us/topic/use-microsoft-translator-in-microsoft-edge-browser-4ad1c6cb-01a4-4227-be9d-a81e127fcb0b), [Firefox](https://support.mozilla.org/en-US/kb/website-translation), [Safari](https://9to5mac.com/2020/12/04/how-to-translate-websites-with-safari-mac/)), e.g. by allowing on-the-fly downloading of different languages instead of assuming all are present from the start.
* Allow a variety of implementation strategies, including on-device vs. cloud-based translation, while keeping these details abstracted from developers.

The following are explicit non-goals:

* We do not intend to force every browser to ship language packs for every language combination, or even to support translation at all. It would be conforming to implement this API by always returning `"no"` from `canTranslate()`, or to implement this API entirely by using cloud services instead of on-device translation.
* We do not intend to provide guarantees of translation quality, stability, or interoperability between browsers. These are left as quality-of-implementation issues, similar to the [shape detection API](https://wicg.github.io/shape-detection-api/). (See also a [discussion of interop](https://www.w3.org/reports/ai-web-impact/#interop) in the W3C "AI & the Web" document.)

The following are potential goals we are not yet certain of:

* Allow web developers to know whether translations are done on-device or using cloud services. This would allow them to guarantee that any user data they feed into this API does not leave the device, which can be important for privacy purposes. (Similarly, we might want to allow developers to request on-device-only translation, in case a browser offers both varieties.)
* Allow web developers to know some identifier for the translation model in use, separate from the browser version. This would allow them to allowlist or blocklist specific models to maintain a desired level of quality.

Both of these potential goals are potentially detrimental to interoperability, so we want to investigate more how important such functionality is to developers to find the right tradeoff.

## Examples

Note that in this API, languages are represented as [BCP 47](https://www.rfc-editor.org/info/bcp47) language tags, as already used by the existing JavaScript `Intl` API or the HTML `lang=""` attribute. Examples: `"ja"`, `"en"`, `"de-AT"`, `"zh-Hans-CN"`.

See [below](#language-tag-handling) for more on the details of how language tags are handled in this API, and the [appendix](#appendix-converting-between-language-tags-and-human-readable-strings) for some helper code that converts between language tags and human-readable strings.

### For a known source language

If the source language is known, using the API looks like so:

```js
const canTranslate = await translation.canTranslate({
  sourceLanguage: "en",
  targetLanguage: "ja"
});

if (canTranslate !== "no") {
  const translator = await translation.createTranslator({
    sourceLanguage: "en",
    targetLanguage: "ja"
  });

  console.assert(translator.sourceLanguage === "en");
  console.assert(translator.targetLanguage === "ja");

  const text = await translator.translate("Hello, world!");
  const readableStreamOfText = await translator.translateStreaming(`
    Four score and seven years ago our fathers brought forth, upon this...`);
} else {
  // Use alternate methods
}
```

### For an unknown source language

If the source language is unknown, the same APIs can be called without the `sourceLanguage` option. The return type of the resulting translator object's `translate()` and `translateStreaming()` methods will change to include the best-guess at the detected language, and a confidence level between 0 and 1:

```js
const canTranslate = await translation.canTranslate({ targetLanguage: "ja" });

if (canTranslate !== "no") {
  const translator = await translation.createTranslator({ targetLanguage: "ja" });

  console.assert(translator.sourceLanguage === null);
  console.assert(translator.targetLanguage === "ja");

  const {
    detectedLanguage,
    confidence,
    result
  } = await translator.translate(someUserText);

  // result is a ReadableStream
  const {
    detectedLanguage,
    confidence,
    result
   } = await translator.translateStreaming(longerUserText);
}
```

If the language cannot be detected, then the return value will be `{ detectedLanguage: null, confidence: 0, result: null }`.

### Downloading new languages

In the above examples, we're always testing if the `canTranslate()` method returns something other than `"no"`. Why isn't it a simple boolean? The answer is because the return value can be one of three possibilities:

* `"no"`: it is not possible for this browser to translate as requested
* `"readily"`: the browser can readily translate as requested
* `"after-download"`: the browser can perform the requested translation, but only after it downloads appropriate material.

To see how to use this, consider an expansion of the above example:

```js
const canTranslate = await translation.canTranslate({ targetLanguage: "is" });

if (canTranslate === "readily") {
  const translator = await translation.createTranslator({ targetLanguage: "is" });
  doTheTranslation(translator);
} else if (canTranslate === "after-download") {
  // Since we're in the "after-download" case, creating a translator will start
  // downloading the necessary language pack.
  const translator = await translation.createTranslator({ targetLanguage: "is" });

  translator.ondownloadprogress = progressEvent => {
    updateDownloadProgressBar(progressEvent.loaded, progressEvent.total);
  };
  await translator.ready;
  removeDownloadProgressBar();

  doTheTranslation(translator);
} else {
  // Use alternate methods
}
```

Note that `await translator.ready` is not necessary; if it's omitted, calls to `translator.translate()` or `translator.translateStreaming()` will just take longer to fulfill (or reject). But it can be convenient.

If the download fails, then `downloadprogress` events will stop being emitted, and the `ready` promise will be rejected with a "`NetworkError`" `DOMException`. Additionally, any calls to `translator`'s methods will reject with the same error.

### Language detection

Apart from translating between languages, the API can offer the ability to detect the language of text, with confidence levels.

```js
if (await translation.canDetect() !== "no") {
  const detector = await translation.createDetector();

  const results = await detector.detect("Hello, world!");
  for (const result of results) {
    console.log(result.detectedLanguage, result.confidence);
  }
}
```

If no language can be detected with reasonable confidence, this API returns an empty array.

### Listing supported languages

To get a list of languages which the current browser can translate, we can use the following code:

```js
for (const language of await translation.supportedLanguages()) {
  let text = languageTagToHumanReadable(lang, "en"); // see appendix
  languageDropdown.append(new Option(text, language));
}
```

This method does not distinguish between languages which are available `"readily"` vs. `"after-download"`, because giving that information for all languages at once is too much of a [privacy issue](#privacy-considerations). Instead, the developer must make individual calls to `canTranslate()`, which gives the browser more opportunities to apply privacy mitigations.

## Detailed design

### Full API surface in Web IDL

```webidl
[Exposed=(Window,Worker)]
interface Translation {
  Promise<TranslationAvailability> canTranslate(TranslationLanguageOptions options);
  Promise<LanguageTranslator> createTranslator(TranslationLanguageOptions options);

  Promise<TranslationAvailability> canDetect();
  Promise<LanguageDetector> createDetector();

  Promise<sequence<DOMString>>> supportedLanguages();
};

[Exposed=(Window,Worker)]
interface LanguageTranslator : EventTarget {
  readonly attribute Promise<undefined> ready;
  attribute EventHandler ondownloadprogress;

  readonly attribute DOMString? sourceLanguage;
  readonly attribute DOMString targetLanguage;

  Promise<(DOMString or ResultWithLanguageDetection)> translate(DOMString input);
  Promise<(ReadableStream or StreamingResultWithLanguageDetection)> translateStreaming(DOMString input);
};

[Exposed=(Window,Worker)]
interface LanguageDetector : EventTarget {
  readonly attribute Promise<undefined> ready;
  attribute EventHandler ondownloadprogress;

  Promise<sequence<LanguageDetectionResult>> detect(DOMString input);
};

partial interface WindowOrWorkerGlobalScope {
  readonly attribute Translation translation;
};

enum TranslationAvailability { "readily", "after-download", "no" };

dictionary TranslationLanguageOptions {
  required DOMString targetLanguage;
  DOMString sourceLanguage;
};

dictionary LanguageDetectionResult {
  DOMString? detectedLanguage;
  double confidence;
};

dictionary ResultWithLanguageDetection : LanguageDetectionResult {
  DOMString? result;
};

dictionary StreamingResultWithLanguageDetection : LanguageDetectionResult {
  ReadableStream? result;
};
```

### Language tag handling

If a browser supports translating from `ja` to `en`, does it also support translating from `ja` to `en-US`? What about `en-GB`? What about the (discouraged, but valid) `en-Latn`, i.e. English written in the usual Latin script? But translation to `en-Brai`, English written in the Braille script, is different entirely.

Tentatively, pending consultation with internationalization and translation API experts, we propose the following model. Each user agent has a list of (language tag, availability) pairs, which is the same one returned by `translation.supportedLanguages()`. Only exact matches for entries in that list will be used for the API.

So for example, consider a browser which supports `en`, `zh-Hans`, and `zh-Hant`. Then we would have the following results:

```js
await translator.canTranslate({ targetLanguage: "en" });      // true
await translator.canTranslate({ targetLanguage: "en-US" });   // false

await translator.canTranslate({ targetLanguage: "zh-Hans" }); // true
await translator.canTranslate({ targetLanguage: "zh" });      // false
```

To improve interoperability and best meet developer expectations, we can mandate in the specification that browsers follow the best practices outlined in BCP 47, especially around [extended language subtags](https://www.rfc-editor.org/rfc/rfc5646.html#section-4.1.2), such as:

* always returning canonical forms instead of aliases;
* correctly distinguishing between script support (e.g. `zh-Hant`) from country support (e.g. `zh-TW`); and
* avoiding including redundant script information (e.g. `en-Latn`).

### Downloading

The current design envisions that the following operations will _not_ cause downloads of language packs or other material like a language detection model:

* `translation.canTranslate()`
* `translation.canDetect()`
* `translation.supportedLanguages()`

The following _can_ cause downloads. In all cases, whether or not a call will initiate a download can be detected beforehand by checking the return value of the corresponding `canXYZ()` call.

* `translation.createTranslator()`
* `translation.createDetector()`

After a developer has a `LanguageTranslator` or `LanguageDetector` object created by these methods, further calls are not expected to cause any downloads. (Although they might require internet access, if the implementation is not entirely on-device.)

## Privacy considerations

This proposal as-is has privacy issues, which we are actively thinking about how to address. They are all centered around how sites that use this API might be able to uniquely fingerprint the user.

The most obvious identifier in the current API design is the list of supported languages, and especially their availability status (`"no"`, `"readily"`, or `"after-download"`). For example, as of the time of this writing [Firefox supports 9 languages](https://www.mozilla.org/en-US/firefox/features/translate/), which can each be [independently downloaded](https://support.mozilla.org/en-US/kb/website-translation#w_configure-installed-languages). With a naive implementation, this gives 9 bits of identifying information, which various sites can all correlate.

Some sort of mitigation may be necessary here. We believe this is adjacent to other areas that have seen similar mitigation, such as the [Local Font Access API](https://github.com/WICG/local-font-access/blob/main/README.md). Possible techniques are:

* Grouping language packs to reduce the number of bits, so that downloading one language also downloads others in its group.
* Partitioning download status by top-level site, introducing a fake download (which takes time but does not actually download anything) for the second-onward site to download a language pack.
* Only exposing a fixed set of languages to this API, e.g. based on the user's locale or the document's main language.

As a first step, we require that detecting the availability of translation for a given language pair be done via individual calls to `canTranslate()`. This allows browsers to implement possible mitigation techniques, such as detecting excessive calls to `canTranslate()` and starting to return `"no"`.

Another way in which this API might enhance the web's fingerprinting surface is if translation and language detection models are updated separately from browser versions. In that case, differing results from different versions of the model provide additional fingerprinting bits beyond those already provided by the browser's major version number. Mandating that older browser versions not receive updates or be able to download models from too far into the future might be a possible remediation for this.

Finally, we intend to prohibit (in the specification) any use of user-specific information in producing the translations. For example, it would not be permissible to fine-tune the translation model based on information the user has entered into the browser in the past.

## Alternatives considered and under consideration

### Streaming input support

Although the API contains support for streaming output of a translation, via the `translateStreaming()` API, it doesn't support streaming input. Should it?

We believe it should not, for now. In general, translation works best with more context; feeding more input into the system over time can produce very different results. For example, translating "<span lang="ja">彼女の話を聞いて、驚いた</span>" to English would give "I was surprised to hear her story". But if you then streamed in another chunk, so that the full sentence was "<span lang="ja">彼女の話を聞いて、驚いたねこが逃げた</span>", the result changes completely to "Upon hearing her story, the surprised cat ran away." This doesn't fit well with how streaming APIs behave generally.

In other words, even if web developers are receiving a stream of input (e.g. over the network or from the user), they need to take special care in how they present such updating-over-time translations to the user. We shouldn't treat this as a usual stream-to-string or stream-to-stream API, because that will rarely be useful.

That said, we are aware of [research](https://arxiv.org/abs/2005.08595) on translation algorithms which are specialized for this kind of setting, and attempt to mitigate the above problem. It's possible we might want to support this sort of API in the future, if implementations are excited about implementing that research. This should be possible to fit into the existing API surface, possibly with some extra feature-detection API.

### Flattening the API and reducing async steps

The current design requires multiple async steps to do useful things:

```js
const translator = await translation.createTranslator(options);
const text = await translator.translate(sourceText);

const detector = await translation.createDetector();
const results = await detector.detect(sourceText);
```

Should we simplify these down with convenience APIs that do both steps at once?

We're open to this idea, but we think the existing complexity is necessary to support the design wherein translation and language detection models might not be already downloaded. By separating the two stages, we allow web developers to perform the initial creation-and-possibly-downloading steps early in their page's lifecycle, in preparation for later, hopefully-quick calls to APIs like `translate()`.

Another possible simplification is to make some of the more informational APIs, namely `canTranslate()`, `canDetect()`, and `supportedLanguages()`, synchronous instead of asynchronous. This would be implementable by having the browser proactively load the information about supported languages into the main thread's process, upon creation of the global object. We think this is not worthwhile, though, as it imposes a non-negligible cost on all global object creation.

### Separating language detection and translation

As discussed in [For an unknown source language](#for-an-unknown-source-language), we support performing both language detection and translation to the best-guess language at the same time, in one API. This slightly complicates the `translate()` and `translateStreaming()` APIs, by giving them polymorphic return types.

We could instead require that developers always supply a `sourceLanguage`, and if they want to detect it ahead of time, they could use the `detect()` API.

We're open to this simplification, but suspect it would be worse for efficiency, as it bakes in a requirement of multiple traversals over the input text, mediated by JavaScript code. We plan to investigate whether multiple traversals over the input are necessary anyway according to the latest research, in which case this simplification would probably be preferable.

## Stakeholder feedback

* W3C TAG: to be requested
* Browser engines:
  * Chromium: prototyping behind a flag
  * Gecko: to be requested
  * WebKit: to be requested
* Web developers: Chrome has received private enthusiasm for such an API, and is working on gathering public evidence of such enthusiasm.

## Appendix: converting between language tags and human-readable strings

This code already works today and is not new to this API proposal. It is likely useful in conjunction with this API, for example when building user interfaces.

```js
function languageTagToHumanReadable(languageTag, targetLanguage) {
  const displayNames = new Intl.DisplayNames([targetLanguage], { type: "language" });
  return displayNames.of(languageTag);
}

languageTagToHumanReadable("ja", "en");      // "Japanese"
languageTagToHumanReadable("zh", "en");      // "Chinese"
languageTagToHumanReadable("zh-Hant", "en"); // "Traditional Chinese"
languageTagToHumanReadable("zh-TW", "en");   // "Chinese (Taiwan)"

languageTagToHumanReadable("en", "ja");      // "英語"
```
