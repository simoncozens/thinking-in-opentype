# Introduction to OpenType Programming

*Goal of this workshop*:

* To be able to write elementary layout code in Adobe feature syntax
* To understand the model and principles of OpenType shaping
* To begin to apply these principles to more advanced script requirements

You will need:

* A copy of the [OTLFiddle](https://github.com/simoncozens/otlfiddle/releases) application.

> This also requires you to have the Python “fontTools” library installed, which you can do with either “pip install fontTools” or “easy_install fontTools” from the command line if you don’t have it already.

* A copy of `OpenSans-Dummy.ttf` and `OpenSans-Dummy.fea`
* A copy of `Mirza-Dummy.ttf` and `Mirza-Dummy.fea`
* A tool like [UnicodeChecker](https://earthlingsoft.net/UnicodeChecker/), [UnicopediaPlus](https://github.com/tonton-pixel/unicopedia-plus) or a web site like [UniView](https://r12a.github.io/uniview/) which will help you to paste unfamiliar characters into text.
* [Optionally] A font editor.

Before the workshop you should:

* Download the resources above.
* Install and test OTLFiddle.
* [Optionally] Setup an Arabic keyboard on your computer.
* Have this workbook open in your PDF viewer or editor.

## Introductions

Welcome to the workshop! You can find what we’re going to be covering in the goals statement above. In the first half of this workshop we are going to be focusing more on understanding the fundamentals of how OpenType features work and how they are processed, rather than looking at specific recipes for doing specific tasks. The aim is that this will give you the tools that you need to solve your own problems in the future and that this knowledge will be transferable to more complex situations. In the second half of the workshop we will demonstrate this by applying these fundamentals to implementing some requirements of an Arabic font.

OpenType programming is a big area, and we cannot cover all of it in one hour! In particular, there are a few important areas which we cannot talk about today: 

* creating fonts with language or script specific behavior;
* how we integrate the code that we write into our font production and mastering processes;
* how we can use OpenType layout rules to define glyph categories using the GDEF table;
* how mark positioning and anchors, normally handled by the font editor, are represented in Adobe feature syntax, and how they can be programmed.

If you want more detail on any of these areas, you might find it in my book [Fonts and Layout for Global Scripts](https://simoncozens.github.io/fonts-and-layout/).

## Basic feature coding - substitutions and ligatures

We’re mostly going to be exploring feature programming through example and experimentation. When it comes to programming, just trying things and seeing what happens is the best way to learn! OTLFiddle was created so that you can quickly test out OpenType rules and get feedback on how they are applied to a piece of text.

OpenType rules are normally written in a language that doesn't exactly have a name - it's known variously as "AFDKO" (from "Adobe Font Development Kit for OpenType"), "Adobe feature language", "fea", "feature format", and so on. Other ways of representing rules are available, (and inside the font they are stored in quite a different representation) but this is the most common way that we can write the rules to program our fonts.

We’re going to begin by coding a simple “ff” ligature. Open up OTLFiddle, drop the `OpenSans-Dummy.ttf` font at the top, and paste in the contents of into `OpenSans-Dummy.fea` the editor on the left. (This provides anchor and mark positioning rules for accents).

Then, at the top of the the editor, add the following lines:

```
feature rlig {
  sub f f by f_f;
} rlig;
```

Now hit “Compile”, and type some text in the text bar on the right. Your window should now look something like this:

![OTLFiddle window](typewknd-1.png)

Congatulations! You’ve made your first OpenType rule. Now I would like you to *experiment* by modifying the rule that you’ve just made:

* Do we need all those semicolons?
* Are spaces significant?
* Do we really need to write “rlig” *again* at the end?

How about changing the rule itself. We’ve successfully substituted two glyphs (f f) for one (f_f). Can we substitute:

* One glyph for one?
* More than one glyph for one?
* One glyph for two or more?
* Two glyphs for two?
* A glyph for zero glyphs? (Can you delete a glyph?)

What about using other features rather than `rlig`? Can we substitute glyphs inside a `kern` feature?!

## Glyph Classes and Named Classes

Now let’s write a set of rules to turn lower case vowels into upper case vowels:

```
feature rlig {
  sub a by A;
  sub e by E;
  sub i by I;
  sub o by O;
  sub u by U;
} rlig;
```

That was a lot of work! Thankfully, it turns out we can write this in a more compact way:

```
feature rlig {
    sub [a e i o u] by [A E I O U];
}  rlig;
```

When a class is used in a substitution, corresponding members are substituted on both sides.

*Experiment*: What happens when a glyph class is only used on the left hand side? Try it and find out:

```
feature rlig {
    sub f [a e i o u] by f_f;
}  rlig;
```

### Named Glyph Classes

Some classes we will use more than once, and it's tedious to write them out each time. We can *define* a glyph class, naming a set of glyphs so we can use the class again later:

```
@lower_vowels = [a e i o u];
@upper_vowels = [A E I O U];
```

Now anywhere a glyph class appeared, we can use a named glyph class instead (including in the definition of other glyph classes!):

```
@vowels = [@lower_vowels @upper_vowels];

feature rlig {
  sub @lower_vowels by @upper_vowels;
} rlig;
```

## How OpenType shaping works

While we could now carry on describing the syntax of the feature file language and giving examples of OpenType rules, this would not necessarily help us to transfer our knowledge to new situations - especially when we are dealing with scripts which have more complicated requirements and expectations. To get that transferable knowledge, we need to have a deeper understanding of what we're doing and how it is being processed by the computer.

So we will now pause our experiments with substitution rules, and before we get into other kinds of rule, we need to step back to look at how the process of OpenType shaping works. *Shaping* is the application of the rules and features within a font to a piece of text. (It also includes operations like decomposition and mark reordering that we're also not going to get into today.)

We've seen that we can put *rules* into a *feature*, but it's important to know that there's another level involved too. Inside an OpenType font, rules are arranged into *lookups*, which are associated with features. Although the language we use to write OpenType code is called "feature language", the primary element of OpenType shaping is the *lookup*.

**To really understand OpenType programming, you need to think in terms of lookups, not features**.

![OpenType model](slide-8.png)

I think of the shaping process as being like an old punched-tape computer. (If you know what a Turing machine is, that's an even better analogy.) The input glyphs that are typed by the user are written on the "tape" and then a "read head" goes over the tape cell-by-cell, checking the current lookup matches at the current position.

![OpenType model](slide-9.png)

If it does, the shaper takes the appropriate action (substitution in the cases we have seen so far). It then moves on to the next location. Once it has gone over the whole tape and performed any actions, the next lookup gets a go (we call this "applying" the lookup).

### Explicit lookups

Let's look at our first example again:

```
feature rlig {
  sub f f by f_f;
} rlig;
```

We can see a feature, and we can see a rule, but where's the lookup? When our feature code was translated into the representation inside an OpenType font, the compiler automatically put it into a lookup for us. But we can also write our lookups explicitly, like this:

```
feature rlig {
  lookup rlig_1 {
    sub f f by f_f;
  } rlig_1;
} rlig;
```

In fact, we're going to write our rules like this from now on, and I'd encourage you to do it when you are working on your own font projects. You can also define a lookup separately, and refer to it within a feature, like this:

```
lookup rlig_1 {
    sub f f by f_f;
} rlig_1;

feature rlig {
    lookup rlig_1;
} rlig;
```

What difference does it make?

*Experiment*: Consider the difference between this:

```
feature rlig {
    sub a by b;
    sub b by c;
} rlig;
```

and

```
feature rlig {
    lookup l1 { sub a by b; } l1;
    lookup l2 { sub b by c; } l2;
} rlig;
```

Try each one out in OTLFiddle against the input string "abc". What happened? Can you work out why? (Hint: How many lookups in the feature in each case? How many rules in each lookup?)

### Ordering of rules, lookups and features

The previous experiment taught us something important: *within a lookup, all rules apply simultaneously*. What? How can you apply more than one rule at once? If we go back to our "paper tape" model of shaping, we can see that in this example, out of our three rules, two of them can apply at glyph position 1: (I'm counting "o" as glyph position zero)

![OpenType model](slide-9.png)

We could either apply the "f f -> f_f" rule or "f f i -> f_f_i" here, but (as we can see) the `f f i` rule is chosen.

*Experiment*: Try changing the order of the rules:

```
feature rlig {
    sub f f by f_f;
    sub f f i by f_f_i;
    sub f l by f_l;
} rlig;
```

Does it make a difference?

It turns out that it doesn't! This is because the feature compiler is doing something clever on our behalf - it is ordering the rules longest-first when it compiles them into the font file, so that the longest match always wins. This is *almost* always what you want, but it does make it a little harder to follow what's going on!

What about the ordering of features? Here, too, it is important to think **lookups not features**. We can talk about the *selection* of features which are applied. For the open source Harfbuzz shaping engine, the list of features which are shaped in a run of text looks like this:

* [rvrn](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rvrn)
* [ltra](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-ltra), [ltrm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-ltrm) (for left-to-right texts) OR [rtla](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rtla), [rtlm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rtlm) (for right-to-left texts)
* [frac](https://docs.microsoft.com/en-us/typography/opentype/spec/features_fj#-tag-frac), [numr](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-numr), [dnom](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-dnom)
* [rand](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rand)
* Script shaper specific features (see the list on the left of [this page](https://docs.microsoft.com/en-us/typography/script-development/use) to find the features for your script.)
* [abvm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-abvm), [blwm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-blwm), [ccmp](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-ccmp), [locl](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-locl), [mark](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-mark), [mkmk](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-mkmk), [rlig](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rlig)
* [calt](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-calt), [clig](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-clig), [curs](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-curs), [dist](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-dist), [kern](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-kern), [liga](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-liga), [rclt](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rclt) OR [vert](https://docs.microsoft.com/en-us/typography/opentype/spec/features_uz#-tag-vert)
* Anything the user turned on

Each of these bullet points represents a *stage*. The stages are processed in the order given above, but *within* a stage, features are *not ordered*. What does this mean? Time for another...

*Experiment*: What does the following code do on the string `abc`?

```
feature rlig { sub a by b; } rlig;

feature ccmp { sub b by c; } ccmp;

feature rlig { sub c by d; } rlig;
```

Guess first before trying it out! Will it produce:

* `bcd`
* `bdd`
* `ccd`
* `ddd`
* Something else?

What about this?

```
feature ccmp { sub b by c; } ccmp;

feature rlig { sub a by b; } rlig;

feature rlig { sub c by d; } rlig;

```

Clearly the order of *lookups* is more important than the order of *features*. In fact, OpenType shaping engines use feature names to *select* lookups within a stage, but within the stage, they then run the lookups in the order they are laid out in the font file - which is also the order they are laid out in the feature file:

```
feature rlig {
lookup l0 { sub a by b; } l0;
} rlig;

feature ccmp {
 lookup l1 { sub b by c; } l1;
} ccmp;

feature test {
 lookup l2 { sub A by B; } l2;
} l2 test;

feature rlig {
 lookup l3 { sub c by d; } l3;
} rlig;
```

If the normal features are selected (i.e. the user hasn't done anything clever to turn on or off certain features), then `rlig` and `ccmp` will be processed, but feature `test` will not be. `rlig` and `ccmp` are in the same stage, which means that the lookups will be run in the order they appear: `l0`, `l1`, `l3`.

**Remember: shapers use *features* to gather *lookups*.**

### Lookup Flags

Lookups may have flags which alter the way they operate.

*Experiment*: Let's go back to our original ligature and try it against some new text:

```
feature rlig {
  lookup ff_ligature {
    sub f f by f_f;
  } ff_ligature;
} rlig;
```

The text I want you to try this against is `f̊f`. (Maybe you want to copy and paste that into OTLFiddle.) It is the string "f", COMBINING RING ABOVE (Unicode codepoint U+030A), "f".

In a way it's pretty obvious that this *shouldn't* work, because the "read head" looks for the two glyphs "f" "f", and finds "f" "ringcomb" instead.

But what if we want to ligate some text regardless of accents? When we come to dealing with our Arabic font, we're going to need to form ligatures between the glyphs which form the main stroke (the *rasm*) while ignoring all the dots and diacriticals around it. Here's one way to deal with the situation:

```
feature rlig {
  lookup ff_ligature {
    sub f f by f_f;
    sub f @accents f by f_f;
  } ff_ligature;
} rlig;
```

But... no, that won't work, because then we substitute away the accent, and we want to keep that. Also it will get tedious if there are more than one possible accents which can occur together.

OpenType lets us declare that some glyphs are *base* glyphs while others are *marks* - which glyphs go in which category is normally set in your font editor, although you [can also do it](http://adobe-type-tools.github.io/afdko/OpenTypeFeatureFileSpecification.html#9.b) in feature code. If we want to do the substitution while ignoring the mark glyphs, we can use the lookup flag `IgnoreMarks`. Try this now:

```
feature rlig {
  lookup ff_ligature {
    lookupflag IgnoreMarks;
    sub f f by f_f;
  } ff_ligature;
} rlig;
```

Now we get the ligature *and* the accent.

Other flags you can apply to a ligature (and you can apply more than one) are:

* `RightToLeft` (Only used for cursive attachment lookups in Nastaliq fonts. You almost certainly don't need this.)
* `IgnoreBaseGlyphs`
* `IgnoreLigatures`
* `IgnoreMarks`
* `MarkAttachmentType @class` (This has been effectively superceded by the next flag; you almost certainly don't need this.)
* `UseMarkFilteringSet @class`

`UseMarkFilteringSet` ignores all marks *except* those in the specified class. This will come in useful when you are, for example, repositioning glyphs with marks above them but you don't really care too much about marks below them.

> We won't experiment with these flags now, but will come back to them in our Arabic example.

### Positioning Phase

We've talked a lot about substitution so far, but that's not the only thing we can use OpenType rules for. You will have noticed that in our "paper tape" model of OpenType shaping, we had two rows on the tape - the top row was the glyph names. What's the bottom row?

The shaper proceeds in two phases: first it applies substitution rules, which go in one part of the font, and then it applies *positioning* rules, which go in another.

In the positioning phase, the shaper associates four numbers with each glyph position. These numbers - the X position, Y position, X advance and Y advance - are called a *value record*, and describe how to draw the string of glyphs. The shaper starts by taking the advance width from metrics of each glyph. As designer, we might think of this as the width of the glyph, but when we come to OpenType programming, it's *much* better to think of this as "where to draw the *next* glyph". Similarly the "position" should be thought of as "where this glyph gets shifted." (The X advance only applies in horizontal typesetting and the Y advance only applies in vertical typesetting.)

In feature syntax, these value records are denoted as four numbers between angle brackets.

![Value records](value-records.png)

As well as writing rules to *substitute* glyphs in the glyph stream, we can also write *positioning* rules to adjust these value records. Let's write one now!

*Experiment*: Remove the features you have added in OTLFiddle, and paste in this one:

```
feature kern {
    lookup adjust_f {
        pos f <0 0 200 0>;
    } adjust_f;
} kern;
```

 * Try it on text like `afbffc`. What did it do?
 * Play about with each of the components of the value record. How do they affect the output?
 * What about *negative* values?
 * Try to turn the "f" into an "accent" - with no width of its own, positioned on top of the letter before it. What happens if you type two f's in a row? (Can you work out why?)

### Types of rule

We've seen a variety of substitution rules, as well as a positioning rule. Are there any other kinds of positioning rule? Are there any other rules? As it happens, the whole set of rule types we need to know about are:

* Substitution
    * One to one
    * Many to one (ligature)
    * One to many
    * Reverse chaining (Only used for Nastaliq fonts)
* Positioning
    * Single positioning: `pos glyph <…>;`
    * Pair positioning: `pos glyph1 <…> glyph2;`
* Anchor
* Chain

Some *questions*:

* Why isn't there a many to many substitution rule? (Hint: `IgnoreMarks`.)
* Why do you think a pair positioning rule might be useful?

*Anchor* rules (of which there are four kinds: cursive attachment, mark positioning, mark-to-mark and mark-to-ligature attachment) are for sticking a glyph onto another glyph. Again, unfortunately we don't have time to get into them today.

*Chain* rules, however, are a lot of fun. They allow us to call a lookup at a given position in the glyph stream, if a certain set of conditions are met. Let's try one first, and then we'll explain it later.

*Experiment*: Paste this into OTLFiddle:

```
lookup ff_ligature {
    sub f f by f_f;
} ff_ligature;

feature rlig {
    sub [A E I O U] f' lookup ff_ligature;
} rlig;
```

Now try it on the string `Off off`. What happens? Why?

A chain rule has three parts, two of which are optional: an optional *backtrack* (also known as *precontext*), the (mandatory) *input*, and the optional *lookahead* (also known as *postcontext*). The input sequence (marked with apostrophe `'` characters) also contains a list of lookups to run at each position if the backtrack, input and postcontext sequences match the glyph stream.

Some notes on chain rules:

* Chains which call substitution lookups must start `sub`. Chains which call position lookups must start `pos`.
* You can chain into lookups containing chains (into lookups containing chains…)!
* You can chain zero, one or (with recent versions of fontTools/makeotf) multiple lookups at each input glyph position:

```
sub [A E I O U]
    f' lookup ff_ligature lookup uppercase # Two lookups
    @lowercase'
    [a e i o u]' lookup uppercase
;
```

* There are “inline” forms of chain rules where you specify the rule to chain on the same line, not as a separate lookup:

```
  pos @medial @mark_above SHADDA’ <-30 100 0 0>;
```

* They will mess you up. Don’t use them.

It's OK if you don't understand this yet. We're going to get some practice on how this all fits together in the next section.

## Introduction to Arabic font engineering

Now let's load the `Mirza-Dummy.ttf` font and paste in the rules from `Mirza-Dummy.fea` into OTLFiddle. The rules I've provided do two things: basic Arabic shaping using the `init`/`medi`/`fina` features, and mark positioning. These are things normally done for you by the font editor, so I don't expect you to code them yourself.

While we've said that features are collections of lookups, *sometimes* the shaper does something a bit more clever. Arabic glyphs have up to four possible forms depending on where they appear in the word: isolated, final, medial and initial. You can see all the different forms of the same letter (Arabic letter meem) in the following string: م ممم. The shaper applies the rules in the `fina` feature only to glyphs in final position, the `medi` feature to glyphs in medial position, and the `init` feature to glyphs in initial position. In those features, you would write ordinary substitution rules to turn e.g. `meem-ar` into `meem-ar.fina` - again, this is something that ordinarily your font editor would write for you, but it's worth knowing what's going on under the hood.

Now I have some challenges for you!

* There is a required ligature in Arabic between the glyphs `lam-ar` and `alef-ar`. These should be replaced by `lam_alef.ar` in isolated form, and `am_alef-ar.fina` when in final form. (Alef is "right-joining", meaning it won't connect to anything on its left, so there's no medial form.) Of course, the ligature should work even if there are marks like `kasra-ar` (U+0650) or `fatha-ar` (U+064E) attached to either the lam or the alef.

You will know you have completed this challenge when the input text `لِا سلاَ` is shaped as `[fatha-ar=9@53,-144|​lam_alef-ar.fina=9+559|​seen-ar.init=7+530|​space=6+200|​kasra-ar=0@335,136|​lam_alef-ar=0+447]`.

*Note* that even though Arabic is read and rendered right-to-left, the OpenType glyph stream is in *logical order* i.e. in `لا` the `lam-ar` comes first.

* Similarly the `kaf-ar` `lam-ar` ligature `kaf_lam-ar` needs to work in all four forms (`kaf_lam-ar`, `kaf_lam-ar.init`, `kaf_lam-ar.medi`, `kaf_lam-ar.fina`) - and of course with potential marks.

* Shape the text `پے` (`peh-ar.init` `yehbarree-ar.fina`). Notice that the dots of the peh clash with the bar of the yeh barree. Write a rule which inserts a `kashida-ar` glyph after `peh-ar.init` and `peh-ar.medi` when the next glyph is `yehbarree-ar.fina`.

*Hint*: You can't directly substitute `peh-ar.init yehbarree-ar.fina` with `peh-ar.init kashida-ar yehbarree-ar.fina` - that would be a many-to-many substitution which OpenType doesn't support. You're going to need to split this into two rules: one which checks that the context is right and chains to another rule, which makes a one-to-many substitution.

* Shape the text `سبِے بِے سیِے یِے`. Notice that the sequence `[beh-ar.medi beh-ar.init yeh-farsi.init yeh-farsi.medi] kasra-ar yehbarree-ar.fina` makes the `kasra-ar` clash into the `yehbarree-ar.fina`. Add a chained positioning rule in the `kern` feature to reposition kasra in this context below the yeh barree.

* Bonus challenge (to take home): Aya ("verse") numbers in the Quran are *enclosed* in a decorative border. When one- to three-digit sequences of Arabic numbers (`one-ar two-ar ...`) are followed by U+06DD END OF AYA SIGN, they should be replaced by small numbers (`one-ar.small two-ar.small ...`). You will need a chained substitution rule to achieve this. They should *also* have a chained *positioning* rule which reduces the advance width of each number to zero, and then displaces them so that they appear centered inside the `endofayah-ar`. Do this with a one-digit number first before working up to two and three digits!

## Close-out

Thank you for attending the workshop. This is the first time it has been run, so please let me know of any feedback you have. I hope it has been useful for you.
