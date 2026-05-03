The explorer covers 24 IPA phonemes across vowels, stops, nasals, fricatives, and approximants — all mapped to ARTIC-48 S-layer channels organized by articulator group (jaw, lips, tongue, velum/larynx, secondary).

A few design decisions worth noting:

**Grouped by articulator, not channel index.** Channels S01–S24 are rendered in anatomical clusters rather than a flat list, so you can immediately see which articulators are active for a given phoneme. A velar stop like `/k/` lights up only the tongue dorsum group; a bilabial `/p/` is entirely a lip/jaw event.

**Bipolar channels** (`larynx_height`, `jaw_lateral`, etc.) render from a center origin leftward for negative values, rightward for positive — the bar direction encodes sign, not just magnitude.

**Zero suppression.** Channels at 0.00 are shown but visually muted, letting you distinguish "not active" from "slightly active" without hiding the full channel set.

To extend this toward P/E/R layers or coarticulation curves across a word, say the word (e.g. `/h ɛ l oʊ/`) and I can generate time-stepped channel data with transition dynamics between phoneme targets.
