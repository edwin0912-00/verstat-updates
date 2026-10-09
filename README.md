# verstat-updates

This GitHub Pages repository serves Sparkle update feeds for tinyVoice and Variator. It contains appcast metadata, not the applications' source code.

| App | Feed | State in this checkout |
| --- | --- | --- |
| tinyVoice | [appcast.xml](https://edwin0912-00.github.io/verstat-updates/tinyvoice/appcast.xml) | Feed exists, with no release entries yet. |
| Variator | [appcast.xml](https://edwin0912-00.github.io/verstat-updates/variator/appcast.xml) | Head entry is version 0.4.0, build 968; it links the full archive and signed deltas. |

The served files are [`tinyvoice/appcast.xml`](tinyvoice/appcast.xml) and [`variator/appcast.xml`](variator/appcast.xml). Variator's appcast carries a Sparkle signature: any content change requires regenerating and re-signing it before publishing.
