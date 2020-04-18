---
title: Finding malicious WebAssembly with yara
date: 2020-02-26T10:00:40Z
draft: false
author: 
 - Dave
summary: I wrote a pattern matcher in rust which runs in Wasm to detect 👻 Wasm
---

tl;dr: I wrote a [pattern matcher][yara-rs] in rust which runs in Wasm to detect
👻 Wasm.

WebAssembly (Wasm) is the new cool, but we're running pre-compiled binaries in
the browser now. What does that mean for detecting and responding to bad stuff?

Wasm does a great job of using a [sandboxed environment][sandbox]. But what if
the malicious code executes happily within the constraints of the sandbox, such
as cryptocurrency mining. This kind of malicious mining activity has received a
bit of attention of the past couple of years as Wasm has become more common.

## How do we detect malicious files elsewhere? ##

On macOS Apple have a have a Malware Removal Tool, XProtect, which scans files
looking for known patterns and removes the files. This uses a [signature
file][signatures] in the [yara][yara] format. 

yara is a "pattern matching swiss knife for malware researchers". It allows
researchers to describe characteristics of a known malicious file and provides a
tool (also called `yara`) which can be used to test if a file matches one of
these descriptions.

So it's both a language for describing characteristics of a file, known as
"rules", and a tool which compares files against these rules looking to see if
they match.

Here's an example rule (taken from XProtect) to better understand what `yara`
is doing.

```yara
rule GenieoD
{
    meta:
        description = "OSX.Genieo.D"
    strings:
        $a = {49 89 C4 0F 57 C0 0F 29 85 80 FE FF FF 0F 29 85 70 FE FF FF 0F 29 85 60 FE FF FF 0F 29 85 50 FE FF FF 41 B8 10 00 00 00 4C 89 E7 48 8B B5 40 FE FF FF 48 8D 95 50 FE FF FF 48}
        $b = {F2 0F 59 C1 F2 0F 5C D0 F2 0F 11 55 B8 0F 28 C2 F2 0F 10 55 D8 F2 0F 10 5D C8 F2 0F 58 DA F2 0F 59 D1 F2 0F 5C DA F2 0F 11 5D B0 0F 28 CB 31 FF BE 05 00 00 00 31 D2}
        $c = {49 6E 73 74 61 6C 6C 4D 61 63 41 70 70 44 65 6C 65 67 61 74 65}
    condition:
        ($a or $b) and $c
}
```

It's split into 3 sections:

 - `meta` is an collection of arbitrary key-value fields (I've seen this used
   for things like setting a severity level to different rules)
 - `strings` includes the patterns which can be found within the malware file
 - `condition` allows us to specify some boolean logic to the patterns
 
In order to determine if a file matches the rule, yara searches the file the
patterns in `strings` then applies the logic from `condition` to evaluate if
it's a match.

There are a lot of ways we should try and detect malicious behaviour in our
systems. Signature-based malware detection is just one of them. I'm not
suggesting this is a path browser vendors or anyone should pursue.

## Running yara in the browser ##

This gave me the idea to try and run `yara` in the browser. This way if a
malicious cryptocurrency miner attempts to load I can detect it and block it.

The `yara` tools are written in C/C++. Libraries exist for other languages but
they are mostly bindings against yaralib. Since I was only interested in a
simple proof-of-concept I decided to try and create my own `yara`
implementation.

## Parsing yara ##

Honestly I'd been looking to write my own parser for something since following
[this excellent tutorial][parser-tutorial] on parser combinators in Rust.
Everything up to this point is me justifying to myself what kind of parser to
create.

That same tutorial recommended the [pom][pom] parser combinator library. Which I
used to create my [yara parser][yara-parser]. It didn't take long at all to get
a working parser for simple yara files. There are some features I'm ignoring for
now though, in order to keep it simple and get a proof of concept working.

Here's an example of a yara rule and the parser output.

```yara
rule add
{
    meta:
        description = "Add example wasm file"
    strings:
        $a = { 00 61 73 6D }
        $b = { 61 64 64 }
    condition:
        ($a and $b)
}
```

```rust
YaraRule {
    name: Str("add",),
    sections: [
        Meta({Str("description",): Str("Add example wasm file",),},),
        Strings({
                Str("$b",): Hex([Byte(54,), Byte(49,), Byte(54,), Byte(52,), Byte(54,), Byte(52,),],),
                Str("$a",): Hex([Byte(48,), Byte(48,), Byte(54,), Byte(49,), Byte(55,), Byte(51,), Byte(54,), Byte(68,),],),
                },),
        Condition(
            And(Identifier(Str("$a",),), Identifier(Str("$b",),),),
        ),
    ],
},
```

It was really interesting to build the parser using combinators, the ways the
pom library uses operators `+ - *` to connect the parsers is really powerful but
looking back at the code it does take a bit of time to figure out what's
happening.

## Simple matching ##

For my proof of concept I wanted byte string and hex (seen in the example above)
matching to be working.

I'm not sure how the real `yara` tool performs these matches but the features in
the hex string matching (wildcards, jumps) made me think to create some kind of
finite state machine because it seemed so similar to regular expressions.

Before trying that I thought may be I could try compiling the hex strings into
regular expressions, then converting the body of the file I was matching into a
hex string and look for matches that way. It seems like an [ugly
solution][matching] but I got something roughly working really quickly.

## Creating a proof of concept ##

My goal was to have make the following code which downloads and instantiates a
malicious cryptocurrency miner fail:

```javascript
fetch('/cryptonight.wasm')
  .then(r => r.arrayBuffer())
  .then(bytes => WebAssembly.instantiate(bytes, {}))
```

The good news is, this code fails already! Job done. Wait, what?..

It seems like -- in Firefox at least -- there is a list of filenames which
cannot be `fetch`ed. I'm not sure where that list is, but `cryptonight.wasm` is
on it. Maybe I missed something else blocking the request, but I could only get
it to `fetch()` once I changed the filename. If you *do know* and could point me
in the direction of the code which creates this behaviour I'd appreciate it.

Anyway, if we `mv cryptonight.wasm nightcrypto.wasm` it now loads successfully.

So now we want to block `nightcrypto.wasm` from loading with our `yara-rs`
tools.

Here's what I ended up with:

```javascript
import init, { yara_match } from '../pkg/yara_rs.js';
async function run() {
  await init();

  let instantiate = WebAssembly.instantiate;
  WebAssembly.instantiate = function(bytes, imports) {
    let view = new Uint8Array(bytes);
    let match = yara_match(`
 rule cryptonight
 {
   meta:
     description = "Crytonight Miner"
   strings:
     $a = { 00 61 73 6D }
     $b = { 63 72 79 70 74 6F 6E 69 67 68 74 5F 68 61 73 68 }
   condition:
     $a and $b
 }`, view);
    if (match) {
      throw new Error("Matched yara Rule");
    } else {
      instantiate(bytes, imports);
    }
  }

  fetch('/sample-miners/nightcrypto.wasm')
    .then(r => r.arrayBuffer())
    .then(bytes => WebAssembly.instantiate(bytes, {}));
}
run();
```

[Give the demo a try](/demo), you'll need to open the developer console to see
yara-rs match the rule and trust that the whole post isn't an elaborate ruse to
get you mining cryptocurrency for me.

If you'd like to try it out for yourself please clone [`yara-rs`][yara-rs] and
follow the instructions in the README. I'm toying with developing a browser
extension to build on this idea a bit further. I'd love to hear from anyone
interested.


[yara-rs]: https://github.com/davbo/yara-rs
[sandbox]: https://webassembly.org/docs/security/
[signatures]: https://gist.github.com/pedramamini/c586a151a978f971b70412ca4485c491
[yara]: https://virustotal.github.io/yara/
[parser-tutorial]: https://bodil.lol/parser-combinators/
[pom]: https://crates.io/crates/pom
[yara-parser]: https://github.com/davbo/yara-rs/blob/627e6c142423855092ecb479d19ce9fc7063b3b2/src/yara/parser.rs
[matching]: https://github.com/davbo/yara-rs/blob/627e6c142423855092ecb479d19ce9fc7063b3b2/src/yara/matcher.rs#L8-L21

