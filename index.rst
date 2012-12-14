========================================
 Bytes, Characters, Codecs, and Strings
========================================

.. note::

   This document was prepared for presentation to the engineering team
   at Eventbrite. It is currently under development; please send
   suggestions to ``nathan+i18n@yergler.net``.

If you've worked with Python 2 for long, you've undoubtedly
encountered Unicode strings. Whether you've known what the little
``u`` preceding the string declaration actually meant -- and more
importantly what it meant you had to do when you started working with
that variable -- is another question. For the first few years I worked
with Unicode in Python, I knew just enough to keep things from blowing
up, but I didn't really understand the underlying concepts or data
structures. The goal of this document is to help engineers understand
Unicode in Python, so they can write code confidently and test for the
sort of cases users are likely to encounter.

.. note::

   Throughout this document, when I refer to "Python", I'm referring
   to Python 2.6. Python 3 attempts to simplify some things related to
   handling Unicode. This is covered later, and when I refer to Python
   3, I'll explicitly mention the version.


An Introduction to Strings
==========================

First, some context. Python has two types of strings: byte strings and
Unicode strings. When we put a sequence of letters in quotes, we're
usually getting a byte string.

  >>> name = "Nathan"
  >>> name
  'Nathan'
  >>> print name
  Nathan

Literally, a sequence of bytes. When you type characters that can't be
expressed in a single byte, you'll see they're stored differently than
they're displayed, and this is where it gets interesting.

  >>> multibyte = "ç"
  >>> multibyte
  '\xc3\xa7'
  >>> print multibyte
  ç

If you prefix the quotes with a lowercase ``u``, Python creates a
Unicode string. A Unicode string is a sequence of characters, or more
specifically Unicode code points.

  >>> unicode_name = u"Nathan"
  >>> unicode_name
  u'Nathan'
  >>> print unicode_name
  Nathan

Of course you can use the extended characters with Unicode strings,
too, but you'll notice they look a little different.

  >>> umultibyte = u"ç"
  >>> umultibyte
  u'\xe7'
  >>> print umultibyte
  ç


A bit of history:

* ASCII was standardized in 1968: 7 bits per character, 127 characters
* By the 1980s, almost all personal computers were 8 bits, so you had
  an additional 128 characters to work with. Different manufacturers
  and different operating systems assigned different characters to
  these extras positions, but that's still only 255 characters in a
  coding system -- not a lot.
* The code pages worked fine for some applications -- if you were
  writing a French document you'd use Latin-1, and if you were writing
  in Russian you'd you KOI8, but what if your French document needed
  to quote Russian?
* Unicode started out with 16 bit characters instead of 8, but has
  since expanded the code space.

Unicode defines characters, called code points, for each character or
glyph. These have names and numbers, which you can use to address
specific code points.

  >>> codepoint = u"\N{ETHIOPIC SYLLABLE WI}"
  >>> codepoint
  u'\u12ca'
  >>> print codepoint
  ዊ

Note that you can only address characters by name in Unicode strings.

  >>> print u"\N{GREEK CAPITAL LETTER DELTA}"
  Δ
  >>> print "\N{GREEK CAPITAL LETTER DELTA}"
  \N{GREEK CAPITAL LETTER DELTA}


So is a character the same as a byte? Not example. Some bytes are
characters and some characters are a single byte, but not all bytes
are characters and not all characters are a single byte.

.. note::

   Not every Unicode code point requires the same amount of space to
   represent: some fit in one byte, some require up to four. Prior to
   Python 3.3, Python has a fixed size it allocated for each character
   in a Unicode string, which was defined at compile time.

   When dealing with C extension modules, this is an issue: an
   extension module compiled to expect 2 byte Unicode (UCS-2) won't
   work with an interpreter compiled for 4 byte Unicode (UCS-4).

   You can tell which your site uses by looking at
   ``sys.maxunicode``::

      >>> import sys
      >>> sys.maxunicode
      65535

   Those of you into Python packaging may remember the brief
   flirtation with binary eggs: this mismatch was a huge pain point
   with deploying binary eggs.

Comparing Unicode & Byte Strings
================================

Comparisons of Unicode and byte strings are sort of interesting;
sometimes it works, sometimes it doesn't.

  >>> name == unicode_name
  True
  >>> multibyte == umultibyte
  False

The latter comparison actually throws a ``UnicodeWarning``, which
we'll talk about later.

Both Unicode and byte strings are immutable: when you create the
instance in memory, it's fixed; any reassignment or alteration will
create a new string object. This means you can safely use string
objects as default values for keyword arguments, or as class level
attributes.


Converting between String types
===============================

Python provides a rich and dynamic type system that tries to stay out
of your way by implicitly converting types. For example, when you add
an integer and a float together, the integer is first converted to a
floating point value. This conversion happens according to the
`type hierarchy`_.

This type hierarchy applies when it comes to string types, as well::

  >>> "Bytes" + " and " +  u"Unicode"
  u"Bytes and Unicode"

It also happens when you perform string formatting, if any of the
participants are a Unicode string::

  >>> "%s and %s" % ("Bytes", u"Unicode")
  u"Bytes and Unicode"

  >>> u"%s and %s" % ("Bytes", "Unicode")
  u"Bytes and Unicode"

But what if you want to explicitly convert between byte and Unicode
strings? The classes for Unicode and byte strings, ``unicode`` and
``str``, also provide support for explicit casts from one type to
another::

  >>> str(u"Unicode")
  "Unicode"

  >>> unicode("Bytes")
  u"Bytes"

.. warning::

   Calling these directly is **not recommended**: in almost all cases
   it results in brittle code.

Since Python 2.3, both ``str`` and ``unicode`` subclass the abstract
type ``basestring``. ``basestring`` provides two methods for
explicitly converting between string types: ``encode``, and
``decode``.

Calling ``encode`` on any string type results in a byte-encoded
string::

  >>> u"Unicode".encode()
  "Unicode"

  >>> "Not Unicode".encode()
  "Not Unicode"

Conversely, calling ``decode`` results in a Unicode string::

  >>> "Not Unicode".decode()
  u"Not Unicode"

  >>> u"Unicode".decode()
  u"Unicode"

But if not all bytes map to a single character, and byte strings may
contain *encoded* characters, how does Python go about handling that
conversion? The answer is **Codecs**.

Generally speaking, a codec is a Python class that can encode Python
Unicode characters to bytes and decode them back to Unicode
characters. Python ships with a set of `standard encodings`_,
including ASCII, UTF-8, UTF-16, and Latin-1. These are available in
the cunningly named codecs_ package. Codecs also includes a set of
"artificial" codecs for encoding and decoding formats such as base-64.

Callings ``encode`` or ``decode`` is the equivalent of asking Python
to load a particular codec and use its ``encode`` or ``decode``
method. You can ask for a specific codec by specifying it by name or
class::

  >>> "Encoded Bytes".decode('utf-8')
  u"Encoded Bytes"

Of course, not all codecs can encode all characters, and bytes encoded
using one codec can not be reliably decoded using a different codec.
Take our earlier multibyte example.

  >>> multibyte
  '\xc3\xa7'
  >>> unicode(multibyte)
  UnicodeDecodeError: 'ascii' codec can't decode byte 0xc3 in position 0: ordinal not in range(128)
  >>> multibyte.decode()
  UnicodeDecodeError: 'ascii' codec can't decode byte 0xc3 in position 0: ordinal not in range(128)
  >>> multibyte.decode('utf8')
  u'\xe7'
  >>> multibyte.decode('utf8') == umultibyte
  True

The Default Encoding
--------------------

When you omit the codec, Python uses the system default codec. When
you install Python, this is depressingly set to the lowest common
denominator::

  >>> import sys
  >>> sys.getdefaultencoding()
  'ascii'

So if there's a ``getdefaultencoding`` is there also a
``setdefaultencoding``?

.. code::

  >>> sys.setdefaultencoding
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  AttributeError: 'module' object has no attribute 'setdefaultencoding'

That's sort of a bummer. But if we look in the Python library
documentation, we see ``setdefaultencoding`` is clearly listed there.

A brief digression into Python startup
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When Python starts up, it imports a few modules. One of these is
``site.py``. `site.py`_ is responsible for setting up the Python
"site", or installation. It does a few things, including adding the
paths for ``site-packages`` and loading any ``pth`` files. The final
step is loading ``sitecustomize.py``, loading ``usercustomize.py``
(new in Python 2.6), and, the answer to our mystery::

    # Remove sys.setdefaultencoding() so that users cannot change the
    # encoding after initialization.  The test for presence is needed when
    # this module is run as a script, because this code is executed twice.
    if hasattr(sys, "setdefaultencoding"):
        del sys.setdefaultencoding

So you *can* customize the default encoding for your Python site, but
you need to do it in ``sitecustomize.py`` or ``usercustomize.py`` (if
enabled). Changing the default encoding after initialization is
considered unsafe.

Implicit Encodes and Decodes
----------------------------

From what we've seen, it *looks* like if your application is working
with a single codec -- say, UTF-8 -- you can safely call
``.decode('utf-8')`` and ``.encode('utf-8')`` on anything to get the
type you want. Not exactly. Consider the following example.

  >>> delta = u"\N{GREEK CAPITAL LETTER DELTA}"
  >>> print delta
  Δ
  >>> delta.encode()
  UnicodeEncodeError: 'ascii' codec can't encode character u'\u0394' in position 0: ordinal not in range(128)
  >>> delta.encode('utf8')
  '\xce\x94'
  >>> delta.decode('utf8')
  UnicodeEncodeError: 'ascii' codec can't encode character u'\u0394' in position 0: ordinal not in range(128)

So what's going on here? We're calling ``decode`` -- which should give
us a Unicode string -- on an existing Unicode string. And we're
specifying a codec that we know can handle the contents of the string.
There are two interesting things about this exception:

* It's an *Encoding* error, when we're trying to *decode*
* It refers to ASCII, when we've clearly specified UTF-8

The problem is this: ``decode`` only operates on byte strings. If it's
called on something other than a byte string, it has to get it to
bytes first. How does it get to bytes? By encoding, of course [I'm
lying a little bit here, but not much.] And what's the default
encoding? ASCII!

So this is a case where calling ``decode`` actually contains an
implicit call to ``encode``, not what you want.


Dealing with Errors
-------------------

The ``encode`` and ``decode`` methods also take an ``errors``
argument. This allows you to customize how errors are dealt with
during encoding and decoding.

The default value of ``errors`` is ``strict``, which throws an
exception when encountering data that can not be encoded or decoded.
Other options include ``replace`` and ``ignore``.

  >>> delta = u"\N{GREEK CAPITAL LETTER DELTA}"
  >>> print delta
  Δ
  >>> delta.encode()
  UnicodeEncodeError: 'ascii' codec can't encode character u'\u0394' in position 0: ordinal not in range(128)
  >>> delta.encode(errors='ignore')
  ''
  >>> delta.encode(errors='replace')
  '?'


Strings and Objects
===================

Thus far we've dealt only with string types. What about handling
conversion to byte strings or Unicode strings for other types? Most
Python developers are familiar with the special method ``__str__``.
Any Python class can define ``__str__`` to provide a byte string
representation of itself. Types can also define a ``__unicode__``
method which should return a Unicode string.

Note that calling ``decode`` on a string object implicitly calls
``__unicode__``. This allows sub-classes to modify the encoding and
decoding behavior.

.. note::

   In addition to ``__str__`` and ``__unicode__``, Python uses
   ``__repr__`` to provide a "representation" of an object.
   ``__repr__`` is used in contexts such as logging where it is *very
   important* that it not raise an exception.

If you want to provide robust support for custom encoding and decoding
behavior, your subclass should also override __add__ and __radd__,
which are called for concatentation, and ``__mod__`` -- literally the
modulus operator -- which is called for string formatting.


Best Practices
==============

* Use ``encode`` and ``decode`` with explicit encodings
* Avoid implicit encodes and decodes that occur when calling ``str``,
  ``unicode``, or calling ``encode`` on a byte string.
* Use ``django.utils.encoding.smart_str``, ``smart_unicode`` from
  within Django



.. best practices
.. smart_str
.. smart_unicode

.. reading data from the wire
.. reading data from the database
.. sending unicode over the wire
.. codecs, files, and the file system


.. Python 2 vs 3

.. _`type hierarchy`: http://docs.python.org/2/reference/datamodel.html#the-standard-type-hierarchy
.. _codecs: http://docs.python.org/2/library/codecs.html
.. _`standard encodings`: http://docs.python.org/2/library/codecs.html#standard-encodings
.. _`site.py`: http://docs.python.org/2/library/site.html
