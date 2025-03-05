---
author: "Marco Pessotto"
date: 2025-03-05
title: "Handling text encoding in Perl"
tags:
 - perl
 - web
 - encoding
 - utf8
 - unicode
---

When we are dealing with legacy applications, it's very much possible
that the code we are looking at does not have a clear understanding of
the problems related to plain text.

In this article we are going to focus on Perl, but the other languages
faces the same problems.

So, in 2025, after more than 30 years since the creation of Unicode,
how is that possible that our application survived ignoring the whole
issue?

Well, if your audience is mainly English speaking, it's possible that
you just experiences glitches sometimes, with some characters like
typographical quotes, non breaking spaces, etc. which are not really
mission-critical.

If, on the contrary, you need to deal every day with diacritics or
even different languages (say, Italian and Slovenian), your
application simply won't survive without a good understanding of
encoding.

### Back to the bytes

As we know, machines work with bytes. A string of text is made of
bytes, and each of them is 8 bits (0 or 1). So one byte allows 256
possible combinations (called codepoints).

Plain ASCII is made by 128 characters, so it fits nicely in one byte,
leaving room for more.

One character is exactly one byte, and one byte carries a character.

However, ASCII is not enough for most of the languages based on the
Latin alphabet, because usually they use diacritics like these: é à č
ž.

So, the [ISO 8859](https://en.wikipedia.org/wiki/ISO/IEC_8859)
encoding standards appeared, which used the free codepoints not
occupied by the ASCII encoding, still using a single byte for a
character and allowing 256 possible characters. That's better, but not
great. It suffices to handle text in a couple of languages if they
share the same characters, but not more. So there are the various
IS0-8859 encoding standards (8859-1, 8859-2, etc.), one for each group
of related languages (e.g. 8859-1 is for Western Europe, 8859-2 for
Central Europe and so on).

The problem is that if you have a random string, you have to guess
which is the correct encoding. Is this character a "È" or "Č"? You
need to look at the context (which language is this?) or searching for
an encoding declaration. Most important, you are simply not able to
type È and Č in the same plain text document.

So finally came the [Unicode](https://en.wikipedia.org/wiki/) age.
This standard allows for more than a million of characters, which
should be enough. So finally you can type English, Italian, Russian,
Arabic and Emojis all in the same plain text. This is truly great, but
it creates a complication for the programmer, because the assumption
that one byte is one character is not true anymore. Usually the
encoding used is UTF-8, which is backward compatible with ASCII. So if
you have an ASCII text, it is also valid UTF-8. However, everything
which is not ASCII, will take from two to three bytes.

### Into the language and back to the world

Text manipulation is a very common task. So if you need to process a
string, say "ÈČ", like in this document, you should be able to tell
that it is a string with two characters representing two letters. You
want to be able to use regular expression on it, and so on.

Now, if we read it as a string of bytes, we get 4 of them and the
newline, which is not what we want.

Let's see an example:

```perl
#!/usr/bin/env perl

use strict;
use warnings;
use Data::Dumper::Concise;

# sample.txt contains ÈČ

{
    open my $fh, '<', 'sample.txt';
    while (my $l = <$fh>) {
        print $l;
        if ($l =~ m/\w\w/) {
            print "Found two characters\n"
        }
        print Dumper($l);
    }
    close $fh;
}
      
{
    open my $fh, '<:encoding(UTF-8)', 'sample.txt';
    while (my $l = <$fh>) {
        print $l;
        if ($l =~ m/\w\w/) {
            print "Found two characters\n"
        }
        print Dumper($l);
    }
    close $fh;
}
```

This is the output:

```
ÈČ
"\303\210\304\214\n"
Wide character in print at test.pl line 24, <$fh> line 1.
ÈČ
Found two characters
"\x{c8}\x{10c}\n"
```

In the first block the file is read verbatim, without any decoding.
The regular expression doesn't work, we have basically 4 bytes which
don't seem to mean much.

In the second block we decoded the input, converting it in the Perl
internal representation. Now we can use regular expressions and have a
consistent approach to text manipulation.

However, we got a warning:

```
Wide character in print at test.pl line 25, <$fh> line 1
```

That's because we printed something to the screen, but given that the
string is now made by characters (decoded for internal use), Perl
warns us that we need to encode it to bytes (for the outside world). A
wide character is basically a character which needs to be encoded.

This can be done either with:

```perl
use strict;
use warnings;
use Encode;
print encode("utf-8", "\x{c8}\x{10c}\n"
```

Or, better, declaring the global encoding for the standard output:

```perl
use strict;
use warnings;
binmode STDOUT, ":encoding(UTF-8)";
print "\x{c8}\x{10c}\n"
```

So, the golden rule is:

 - decode the string on input and get characters out of bytes
 - work with it in your program as a string of characters
 - encode the string on output
 
### Encoding strategies

If you are dealing with standard input/output on the shell, add this
at the beginning of your script:

```
use strict;
use warnings;
binmode STDIN,  ":encoding(UTF-8)";
binmode STDOUT, ":encoding(UTF-8)";
binmode STDERR, ":encoding(UTF-8)";
```

So you're decoding on input and encoding on output automatically.

For files, you can add the layer in the second argument of `open` like
in the sample script above, or use a more handy module like
`Path::Tiny`, which provides methods like `slurp_utf8` and `spew_utf8`
to get characters out of the files.

The interactions with web frameworks should always happen with the
internal Perl representation. So when you receive the input from a
form, it *should be considered already decoded*. It's also
responsibility of the framework to handle the encoding on output,
modulo bugs of course. Here at End Point we have many Interchange
applications. Interchange *can* support this, via the `MV_UTF8`
variable.

The same rules applies to databases. It's responsibility of the driver
to take your data, encode it when storing it in the database, and
decoding it when retrieving data from the database. E.g.
[DBD::Pg](https://metacpan.org/pod/DBD::Pg) has `pg_enable_utf8` and
[DBD::mysql](https://metacpan.org/pod/DBD::mysql) has
`mysql_enable_utf8`. These options should usually be turned on or off
explicitly. Not specifying the option (even just to disable it) is
usually source of confusion because of the heuristic approach they
take.

### Debugging strategies

It may not be the most correct approach, but that's what I've been
using for more than a decade and works, and it's simply to use
`Data::Dumper` and call `Dumper` on the string you want to examine.

If you see the hexadecimal codepoints like `\x{c8}\x{10c}`, it means
the string is decoded and you're working with the characters, if you
see the raw bytes, you're dealing with an encoded string, so you need
to go upstream and find out what should have decoded it.

### Migrate a web application to Unicode

If you're still using legacy encoding systems like 8859 or the similar
Windows ones, or worse, you simply don't know and you're relying on
the browsers' heuristics (they're quite good at that) you need to
handle the input and the output correctly along the whole perimeter:

 - inspect and possibly convert the existing DB data
 - make sure the DB drivers handle the I/O correctly
 - make sure the Web framework is decoding the input and encoding the output
 - make sure the files you read and write are correctly handled
 - clean up any workaround you may had in place to make it so far
 
This can look, and is, a challenging task, but it's totally worth of
it, as fancy characters nowadays are the norm. Typographical quotes
like "“this” and ‘this’ are very common and inserted by word
processors automatically. So are Emojis. Not being able to use Unicode
also means you are not able to write correctly, e.g., customer notes.

### Band-aids

If your client is on a budget or can't deal with a large upgrade like
this one, which has the potential to be disruptive and exposing bugs
which are lurking around, you can try to degrade the unicode
characters to ASCII with tools like
[Text::Unidecode](https://metacpan.org/pod/Text::Unidecode) (which, by
the way, has been ported to other languages as well). So typographical
quotes will became the ASCII plain ones, diacritics will be stripped,
and even various characters will get their ASCII representation. Not
great, but better than nothing!

