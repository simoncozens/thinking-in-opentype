# Introduction to OpenType Programming

*Goal of this workshop*:

* To be able to write elementary layout code in Adobe feature syntax
* To understand the model and principles of OpenType shaping
* To begin to apply these principles to more advanced script requirements

You will need:

* A copy of the [OTLFiddle](https://github.com/simoncozens/otlfiddle/releases)) application.

> This also requires you to have the Python “fontTools” library installed, which you can do with either “pip install fontTools” or “easy_install fontTools” from the command line if you don’t have it already.

* A copy of `OpenSans-Dummy.ttf` and `OpenSans-Dummy.fea`
* A copy of `Amiri-Dummy.ttf` and `Amiri-Dummy.fea`
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
* [frac]](https://docs.microsoft.com/en-us/typography/opentype/spec/features_fj#-tag-frac), [numr](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-numr), [dnom](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-dnom)
* [rand](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rand)
* Script shaper specific features (see the list on the left of [this page](https://docs.microsoft.com/en-us/typography/script-development/use) to find the features for your script.)
* [abvm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-abvm), [blwm](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-blwm), [ccmp](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-ccmp), [locl](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-locl), [mark](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-mark), [mkmk](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-mkmk), [rlig](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rlig)
* [calt](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-calt), [clig](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-clig), [curs](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-curs), [dist](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ae#-tag-dist), [kern](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-kern), [liga](https://docs.microsoft.com/en-us/typography/opentype/spec/features_ko#-tag-liga), [rclt](https://docs.microsoft.com/en-us/typography/opentype/spec/features_pt#-tag-rclt) OR [vert](https://docs.microsoft.com/en-us/typography/opentype/spec/features_uz#-tag-vert)
* Anything the user turned on

Each of these bullet points represents a *stage*. The stages are processed within the order given above, but *within* a stage, lookups are *not ordered*. What does this mean? Time for another

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

If the normal feature are selected, `rlig` and `ccmp` are on, but feature `test` is not turned on by the user. `rlig` and `ccmp` are in the same stage, which means that the lookups will be run in the order they appear: `l0`, `l1`, `l3`.

**Remember: shapers use *features* to gather *lookups*.**

### Lookup Flags

Lookups may have flags which alter the way they operate.

Experiment: Let's go back to our original ligature and try it against some new text:

```
feature rlig {
  lookup ff_ligature {
    sub f f by f_f;
  } ff_ligature;
} rlig;
```

The text I want you to try this against is `f̊f`. (Maybe you want to copy and paste that into OTLFiddle.) It is the string "f", COMBINING RING ABOVE (Unicode codepoint U+030A), "f".

In a way it's pretty obvious that this *shouldn't* work, because the "read head" looks for the two glyphs "f" "f", and finds "f" "ringcomb" instead.

* Flags - experiment: ff/f̊f

* Phases: Sub and Pos phase

### Types of rule

* Sub - types of sub. Why no many to many?
* Pos - experiment. ???  (Value records)
* Anchor 
* Chain - experiment: swap two letters
		
## Introduction to Arabic font engineering

* Explain init/medi/fina

* Challenges:
  * Lam-Alif ligature (isolated and final)
  * Lam kaf is too close. Insert a kashida between them (init and medi)
  * Daf-Kaf  / Rah-kaf contextual kerning

* Accordion challenges:
  * Repeated beh sequence

## Questions and close out
