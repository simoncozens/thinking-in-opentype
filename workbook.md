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

Congatulations! You’ve made your first OpenType rule. Now I would like you to experiment by modifying the rule that you’ve just made:

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

When a class is used in a substitution, corresponding members are substituted on both sides. What happens when a glyph class is only used on the left hand side? Try it and find out:

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

We've seen that we can put *rules* into a *feature*, but it's important to know that there's another level involved too. Inside an OpenType font, rules are arranged into *lookups*, which are associated with features. **To really understand OpenType programming, you need to think in terms of lookups, not features**.

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

We can see a feature, and we can see a rule, but where's the lookup? 

### The shaper model

* Lookup as primary Experiment -rewrite with lookup. Why you should do that.
* All rules run simultaneously - Longest match within a feature wins - Ordering done for you
* Which features are applied (Language specific)
* Features not ordered, lookups ordered
* Poll for slide 13: bcd / ccd / ddd
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
