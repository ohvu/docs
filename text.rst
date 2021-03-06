Text
====

When discussing text in the context of programming there are a few different views we can take at different levels of abstraction, so let's define some terms.

- *Literal* - A piece of textual data as it appears in source code.

- *String* - A sequence of characters which defines a piece of text in the abstract.
  
- *Character* - The indivisible unit from which a string is composed.

- *Representation* - The s-expressions which represent strings and characters at runtime. Every string must be encodable in Unicode, and all Unicode text must be representable as a string.

- *Layout* - A scheme for storing a representation in computer memory. A typical compiler or runtime implementation may choose to lay out strings according to a standard encoding such as UTF-8 or UTF-32, but this is not a requirement as the choice should not be visible to user code.

Literals
--------

Basic string literals in Calipto are delimited by quotes ``"``::

  "For example, this is a string."

Unlike in most languages, string literals in Calipto do not support escape sequences by default. Instead, if a string is to contain quotes, the delimiters must be made longer so as to distinguish them from the string content::

  ""Thus any quotes within the string are "disambiguated" from the delimiters.""

This means that the literal ``""`` cannot be used to denote an empty string, since it is ambiguous with an opening delimiter. This is not a problem, as the empty string is just an empty list and can be denoted by ``()``.

A string can also span multiple lines::

  "
  This is a multi-line literal denoting a single-line string.
  "

  ""
  And this is a multi-line literal denoting ...
  ... a "multi-line" string.
  ""

In this case, the opening delimiter must be immediately followed by a newline, and every following line must be aligned to it by preceding spaces. The first and last newlines are ignored when present, as are the aligning spaces::

  (uppercase "
             Help!
             Please!
             ")

  (unescape \\
    This is a string!
    \\)

  (unescape \ This is a string with an \inline value\)

Readers may notice that single-line literals cannot begin and end with quotes, as they cannot be distinguished from delimiters. Such strings must be written as multi-line literals::

  """
  "Hello," she said, "I come in peace."

  """

  \""Hello," she said, "I come in peace."\"

  (strip-indent
    \"\
    "Hello," she said, "I come in peace."
    \")

The choice not to support escape sequences or backslashes is justified by the ease with which strings can be concatenated::

  (cat "Hello, string." \n)

  (cat multiplier " times 2.5 is " (#decimal 2.5 multiplier))

  \"\multiplier times 2.5 is \(#decimal 2.5 multiplier))
  \"

  (unescape \ "\multiplier times 2.5 is \(#decimal 2.5 multiplier)\")

  (unescape $ "C:\Users\$(name user)\.app\$")

  (unescape ( ) "C:\Users\(user-name)\.app\")

  (cat "C:\Users\" (name user) "\.app\")

  "\ \multiplier times 2.5 is \(#decimal 2.5 multiplier)) \"

  (unescape
    \"\
    This is the first line...
    ... and this is the second!
    Here's a value in quotes: "\(inline-value)"!
    And now for the final line.
    \")

  (strip-indent (cat \"\
                     This is the first line...
                     ... and this is the second!
                     Here's a value in quotes: "\" (inline-value) \""!
                     And now for the final line.
                     \"))

  (strip-indent (cat ""
                     This is the first line...
                     ... and this is the second!
                     Here's a value in quotes: """ (inline-value) ""\"!
                     And now for the final line.
                     \""))

.. todo::
  
  Maybe we should support escapes? They can be explicitly enabled by following the opening delimiter with a sequence of backslashes.

  (strip-indent (cat ""\
                     This is the first line...
                     ... and this is the second!
                     Here's a value in quotes: "\(inline-value)"!
                     And now for the final line.
                     \""))
  
  In this example an interpolated value is an escape marker followed by whitespace. We don't need special syntax to close the interpolated value, we just call "read" on the scanner at that point and it ends where it ends...

  This combination of rules does make it oddly difficult to start a string with slashes...
  "
  \this line is the string proper\
  "

Strings
-------

A string is simply a sequence of characters, so this is how Calipto represents them. Simply as a proper list with each subsequent element being the subsequent character.

Characters
----------

If a string is a sequence of characters, we need to define exactly what constitutes a "character" and how this notion is represented at runtime. This turns out be be a tricky choice with a number of subtle tradeoffs, as the word character is heavily overloaded.

In the Unicode standard there are a few candidates for what could be considered a character.

- A *glyph* is a specific written representation of a "grapheme", for instance the way a grapheme is rendered according to a choice of font.

- A *grapheme* is what a human reader would typically consider a single character. It may be composed of multiple "code points".

- A *code point* is the unit of information of text in the abstract. It is independent of encoding and may be composed of multiple "code units".

- A *code unit* is the unit of encoding of text as a series of bytes. It is dependent on the chosen encoding format (i.e. UTF-8, UTF-16, UTF-32, etc.)

In Calipto we choose code points as our unit of representation, and hereafter we will interchangeably refer to code points as characters. The justification for this choice is multi-faceted.

Code units are unsuitable to be the character unit, as they are an artifact of the choice of encoding which is generally considered to be an implementation detail. Glyphs, too, are unsuitable to be the character unit, as they are concerned with the manner of representation of text rather than the abstract text itself.

Graphemes are a more attractive target, but unfortunately they are sometimes defined to be locale-dependent in Unicode, and it is also sometimes useful to manipulate text at the sub-grapheme level.

So for these reasons, code points are left as the best option to be our character, and our unit of representation. This has a few other benefits, for instance it aligns with the terminology of the Unicode standard, which assigns "Abstract Characters" to single code points. It also simplifies laying characters out in memory in some circumstances, as all code points can fit into the same fixed amount of space.

However, many text manipulation tasks should operate over graphemes and not "characters", and for this reason many of the operations defined over text are parametric with a strategy for "clustering" graphemes.

Representation
==============

We have defined an abstract notion of a character for our purposes, but we have not discussed how a character is to be represented at runtime. In Calipto we choose to represent each character as a distinct symbol in a special namespace ``unicode``.

The purpose of a symbol is only to describe a unique combination of name and namespace. It is tempting to say that the symbol name for a character should be the character itself, but this is problematic:

- Some symbols are "combining marks" which don't make sense by themselves, for example accents.
  
- Whitespace characters would not appear in some circumstances.

- Many characters are indistinguishable in most fonts.

Instead, the name for a character's symbol is simply the unicode name string.

Layout
======

It might feel unnecessarily "heavy" to burden each character of a string with its entire name, but when processing strings we don't actually have to reflect over the names of the symbols of the constituent characters so we can still lay the characters out in memory efficiently and operate on them quickly. It's typically only for debugging purposes that the system will be asked to fetch the name of a character symbol, at which point it may be useful for the programmer to be given a full name.

Since characters are represented nominally rather than numerically by index, this does preclude use from easily employing arithmetic over characters at the language level. Since the layout is likely to be numeric, however, tricks such as intrinsics can be used to avoid having to materialise huge tables just to encode and decode text, so Calipto should be able to perform these tasks as fast as any language.

Graphemes
---------

TODO discuss grapheme clustering.
Normalisation
TODO discuss normalisation & equivalence (Unicode specifies 4 normal forms!)

Normalisation probably shouldn't be done automatically, as this prohibits round-tripping from external formats.

Newlines are just \n on all platforms by default.

Scanning
--------

Characters in strings cannot be directly indexed, they must be reached and counted by iterating over them. This is to make it easier for language implementations to choose compressed storage strategies such as UTF-8 or UTF-16.

Graphemes can also not be directly indexed or counted. TODO explain why.

In other words, most useful operations over strings are defined iteratively. It turns out that iterative operations are more widely applicable than just to static strings, for instance we may also be able to iterate efficiently over streaming network sources, or text generated dynamically. For this reason we abstract all our important string processing operations into a more general text scanning API.
