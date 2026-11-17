# SFX Analysis Format

An open, plain-text format for storing measurements of every file in a sound-effects library. Analyse your library once. After that, any AI agent or automated tool can build a strong starting pool of sounds without decoding the audio again.

**Status: v0.1 draft.** The specification is here. The reference analyser is published when it passes its known-answer test set.

## Why

Choosing sound effects means listening for things a filename can't tell you:
- how sharp the attack is, and where exactly each hit starts;
- how long the tail rings, and how dry or roomy the take is;
- how bright or heavy it is, whether it's mono-safe, and how loud.

Every tool that wants to help with that measures the same things again and again. This format measures them once, writes them down in documented units, and lets anything read them.

- **Per file:** quality flags, loudness, stereo, space, resonances, a pitch trend, and a 74-number profile of the envelope, colour, tone and texture.
- **Per hit:** every attack front, peak and natural end, in the file's own samples, plus each hit's own profile.
- **In plain words:** a short "sound card" built from the numbers by fixed rules, so any reader can use it without audio tools.

It works alongside the [Universal Category System](https://universalcategorysystem.com). UCS names **what** a sound is; this format measures **how** it sounds and **where** its hits are.

## What it is not

- It holds no audio, and nothing that can rebuild audio.
- It doesn't choose sounds. It feeds a pool of suggestions, and the editing is still yours.
- It's never a shared index of anyone's library. Each user analyses their own. Only the format and the analyser are public.

## Read the spec

[SPEC.md](SPEC.md)

## Licence

MIT. See [LICENSE](LICENSE).

By J. Murphy Ryan ([jmryansound.com](https://jmryansound.com)). Also by the same author: [ucs-tools](https://github.com/jmrsound/ucs-tools).
