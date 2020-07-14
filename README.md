bitarray: efficient arrays of booleans
======================================

This module provides an object type which efficiently represents an array
of booleans.  Bitarrays are sequence types and behave very much like usual
lists.  Eight bits are represented by one byte in a contiguous block of
memory.  The user can select between two representations: little-endian
and big-endian.  All of the functionality is implemented in C.
Methods for accessing the machine representation are provided.
This can be useful when bit level access to binary files is required,
such as portable bitmap image files (.pbm).  Also, when dealing with
compressed data which uses variable bit length encoding, you may find
this module useful.


Key features
------------

 * All functionality implemented in C.
 * Bitarray objects behave very much like a list object, in particular
   slicing (including slice assignment and deletion) is supported.
 * The bit endianness can be specified for each bitarray object, see below.
 * Packing and unpacking to other binary data formats, e.g. numpy.ndarray
   is possible.
 * Fast methods for encoding and decoding variable bit length prefix codes
 * Bitwise operations: `&`, `|`, `^`, `&=`, `|=`, `^=`, `~`
 * Sequential search
 * Pickling and unpickling of bitarray objects.
 * Bitarray objects support the buffer protocol (Python 2.7 and above)
 * On 32-bit systems, a bitarray object can contain up to 2^34 elements,
   that is 16 Gbits (on 64-bit machines up to 2^63 elements in theory).


Installation
------------

Bitarray can be installed from source:

    $ tar xzf bitarray-1.4.1.tar.gz
    $ cd bitarray-1.4.1
    $ python setup.py install

On Unix systems, the latter command may have to be executed with root
privileges.  You can also pip install bitarray.
Once you have installed the package, you may want to test it:

    $ python -c 'import bitarray; bitarray.test()'
    bitarray is installed in: /usr/local/lib/python2.7/site-packages/bitarray
    bitarray version: 1.4.1
    3.7.4 (r271:86832, Dec 29 2018) [GCC 4.2.1 (SUSE Linux)]
    .........................................................................
    .........................................................................
    ..............................
    ----------------------------------------------------------------------
    Ran 230 tests in 0.889s

    OK

You can always import the function test,
and `test().wasSuccessful()` will return `True` when the test went well.


Using the module
----------------

As mentioned above, bitarray objects behave very much like lists, so
there is not too much to learn.  The biggest difference from list
objects (except that bitarray are obviously homogeneous) is the ability
to access the machine representation of the object.
When doing so, the bit endianness is of importance; this issue is
explained in detail in the section below.  Here, we demonstrate the
basic usage of bitarray objects:

    >>> from bitarray import bitarray
    >>> a = bitarray()            # create empty bitarray
    >>> a.append(True)
    >>> a.extend([False, True, True])
    >>> a
    bitarray('1011')

Bitarray objects can be instantiated in different ways:

    >>> a = bitarray(2**20)       # bitarray of length 1048576 (uninitialized)
    >>> bitarray('1001011')       # from a string
    bitarray('1001011')
    >>> lst = [True, False, False, True, False, True, True]
    >>> bitarray(lst)             # from list, tuple, iterable
    bitarray('1001011')

Bits can be assigned from any Python object, if the value can be interpreted
as a truth value.  You can think of this as Python's built-in function bool()
being applied, whenever casting an object:

    >>> a = bitarray([42, '', True, {}, 'foo', None])
    >>> a
    bitarray('101010')
    >>> a.append(a)      # note that bool(a) is True
    >>> a.count(42)      # counts occurrences of True (not 42)
    4
    >>> a.remove('')     # removes first occurrence of False
    >>> a
    bitarray('110101')

Like lists, bitarray objects support slice assignment and deletion:

    >>> a = bitarray(50)
    >>> a.setall(False)
    >>> a[11:37:3] = 9 * bitarray([True])
    >>> a
    bitarray('00000000000100100100100100100100100100000000000000')
    >>> del a[12::3]
    >>> a
    bitarray('0000000000010101010101010101000000000')
    >>> a[-6:] = bitarray('10011')
    >>> a
    bitarray('000000000001010101010101010100010011')
    >>> a += bitarray('000111')
    >>> a[9:]
    bitarray('001010101010101010100010011000111')

In addition, slices can be assigned to booleans, which is easier (and
faster) than assigning to a bitarray in which all values are the same:

    >>> a = 20 * bitarray('0')
    >>> a[1:15:3] = True
    >>> a
    bitarray('01001001001001000000')

This is easier and faster than:

    >>> a = 20 * bitarray('0')
    >>> a[1:15:3] = 5 * bitarray('1')
    >>> a
    bitarray('01001001001001000000')

Note that in the latter we have to create a temporary bitarray whose length
must be known or calculated.


Bit endianness
--------------

Since a bitarray allows addressing of individual bits, where the machine
represents 8 bits in one byte, there are two obvious choices for this
mapping: little- and big-endian.
When creating a new bitarray object, the endianness can always be
specified explicitly:

    >>> a = bitarray(endian='little')
    >>> a.frombytes(b'A')
    >>> a
    bitarray('10000010')
    >>> b = bitarray('11000010', endian='little')
    >>> b.tobytes()
    b'C'

Here, the low-bit comes first because little-endian means that increasing
numeric significance corresponds to an increasing address (index).
So `a[0]` is the lowest and least significant bit, and `a[7]` is the
highest and most significant bit.

    >>> a = bitarray(endian='big')
    >>> a.frombytes(b'A')
    >>> a
    bitarray('01000001')
    >>> a[6] = 1
    >>> a.tobytes()
    b'C'

Here, the high-bit comes first because big-endian
means "most-significant first".
So `a[0]` is now the lowest and most significant bit, and `a[7]` is the
highest and least significant bit.

The bit endianness is a property attached to each bitarray object.
When comparing bitarray objects, the endianness (and hence the machine
representation) is irrelevant; what matters is the mapping from indices
to bits:

    >>> bitarray('11001', endian='big') == bitarray('11001', endian='little')
    True

Bitwise operations (`&`, `|`, `^`, `&=`, `|=`, `^=`, `~`) are implemented
efficiently using the corresponding byte operations in C, i.e. the operators
act on the machine representation of the bitarray objects.
Therefore, one has to be cautious when applying the operation to bitarrays
with different endianness.

When converting to and from machine representation, using
the `tobytes`, `frombytes`, `tofile` and `fromfile` methods,
the endianness matters:

    >>> a = bitarray(endian='little')
    >>> a.frombytes(b'\x01')
    >>> a
    bitarray('10000000')
    >>> b = bitarray(endian='big')
    >>> b.frombytes(b'\x80')
    >>> b
    bitarray('10000000')
    >>> a == b
    True
    >>> a.tobytes() == b.tobytes()
    False

The endianness can not be changed once an object is created.
However, since creating a bitarray from another bitarray just copies the
memory representing the data, you can create a new bitarray with different
endianness:

    >>> a = bitarray('11100000', endian='little')
    >>> a
    bitarray('11100000')
    >>> b = bitarray(a, endian='big')
    >>> b
    bitarray('00000111')
    >>> a == b
    False
    >>> a.tobytes() == b.tobytes()
    True

The default bit endianness is currently big-endian, however this may change
in the future, and when dealing with the machine representation of bitarray
objects, it is recommended to always explicitly specify the endianness.

Unless explicitly converting to machine representation, using
the `tobytes`, `frombytes`, `tofile` and `fromfile` methods,
the bit endianness will have no effect on any computation, and one
can safely ignore setting the endianness, and other details of this section.


Buffer protocol
---------------

Python 2.7 provides memoryview objects, which allow Python code to access
the internal data of an object that supports the buffer protocol without
copying.  Bitarray objects support this protocol, with the memory being
interpreted as simple bytes.

    >>> a = bitarray('01000001' '01000010' '01000011', endian='big')
    >>> v = memoryview(a)
    >>> len(v)
    3
    >>> v[-1]
    67
    >>> v[:2].tobytes()
    b'AB'
    >>> v.readonly  # changing a bitarray's memory is also possible
    False
    >>> v[1] = 111
    >>> a
    bitarray('010000010110111101000011')


Variable bit length prefix codes
--------------------------------

The method `encode` takes a dictionary mapping symbols to bitarrays
and an iterable, and extends the bitarray object with the encoded symbols
found while iterating.  For example:

    >>> d = {'H':bitarray('111'), 'e':bitarray('0'),
    ...      'l':bitarray('110'), 'o':bitarray('10')}
    ...
    >>> a = bitarray()
    >>> a.encode(d, 'Hello')
    >>> a
    bitarray('111011011010')

Note that the string `'Hello'` is an iterable, but the symbols are not
limited to characters, in fact any immutable Python object can be a symbol.
Taking the same dictionary, we can apply the `decode` method which will
return a list of the symbols:

    >>> a.decode(d)
    ['H', 'e', 'l', 'l', 'o']
    >>> ''.join(a.decode(d))
    'Hello'

Since symbols are not limited to being characters, it is necessary to return
them as elements of a list, rather than simply returning the joined string.


Reference
=========

The bitarray object:
--------------------

`bitarray(initializer=0, /, endian='big')` -> bitarray

Return a new bitarray object whose items are bits initialized from
the optional initial object, and endianness.
The initializer may be of the following types:

`int`: Create a bitarray of given integer length.  The initial values are
arbitrary.  If you want all values to be set, use the .setall() method.

`str`: Create bitarray from a string of `0` and `1`.

`list`, `tuple`, `iterable`: Create bitarray from a sequence, each
element in the sequence is converted to a bit using its truth value.

`bitarray`: Create bitarray from another bitarray.  This is done by
copying the memory holding the bitarray data, and is hence very fast.

The optional keyword arguments `endian` specifies the bit endianness of the
created bitarray object.
Allowed values are the strings `big` and `little` (default is `big`).

Note that setting the bit endianness only has an effect when accessing the
machine representation of the bitarray, i.e. when using the methods: tofile,
fromfile, tobytes, frombytes.


**A bitarray object supports the following methods:**

`all()` -> bool

Returns True when all bits in the array are True.


`any()` -> bool

Returns True when any bit in the array is True.


`append(item, /)`

Append the truth value `bool(item)` to the end of the bitarray.


`buffer_info()` -> tuple

Return a tuple (address, size, endianness, unused, allocated) giving the
memory address of the bitarray's data, the size (in bytes) used to hold the
bitarray's contents, the bit endianness as a string, the number of unused
bits in the last bytes, and the size (in bytes) of the allocated memory.


`bytereverse()`

For all bytes representing the bitarray, reverse the bit order (in-place).
Note: This method changes the actual machine values representing the
bitarray; it does not change the endianness of the bitarray object.


`clear()`

Remove all items from the bitarray.


`copy()` -> bitarray

Return a copy of the bitarray.


`count(value=True, start=0, stop=<end of array>, /)` -> int

Count the number of occurrences of bool(value) in the bitarray.


`decode(code, /)` -> list

Given a prefix code (a dict mapping symbols to bitarrays),
decode the content of the bitarray and return it as a list of symbols.


`encode(code, iterable, /)`

Given a prefix code (a dict mapping symbols to bitarrays),
iterate over the iterable object with symbols, and extend the bitarray
with the corresponding bitarray for each symbol.


`endian()` -> str

Return the bit endianness as a string (either `little` or `big`).


`extend(iterable or string, /)`

Extend bitarray by appending the truth value of each element given
by iterable.  If a string is provided, each `0` and `1` are appended
as bits.


`fill()` -> int

Adds zeros to the end of the bitarray, such that the length of the bitarray
will be a multiple of 8.  Returns the number of bits added (0..7).


`frombytes(bytes, /)`

Extend bitarray with raw bytes.  That is, each append byte will add eight
bits to the bitarray.


`fromfile(f, n=-1, /)`

Extend bitarray with up to n bytes read from the file object f.
When n is omitted or negative, reads all data until EOF.
When n is provided and positions but exceeds the data available,
EOFError is raised (but the available data is still read and appended.


`index(value, start=0, stop=<end of array>, /)` -> int

Return index of the first occurrence of `bool(value)` in the bitarray.
Raises `ValueError` if the value is not present.


`insert(index, value, /)`

Insert `bool(value)` into the bitarray before index.


`invert()`

Invert all bits in the array (in-place),
i.e. convert each 1-bit into a 0-bit and vice versa.


`iterdecode(code, /)` -> iterator

Given a prefix code (a dict mapping symbols to bitarrays),
decode the content of the bitarray and return an iterator over
the symbols.


`itersearch(bitarray, /)` -> iterator

Searches for the given a bitarray in self, and return an iterator over
the start positions where bitarray matches self.


`length()` -> int

Return the length, i.e. number of bits stored in the bitarray.
This method is preferred over `__len__` (used when typing `len(a)`),
since `__len__` will fail for a bitarray object with 2^31 or more elements
on a 32bit machine, whereas this method will return the correct value,
on 32bit and 64bit machines.


`pack(bytes, /)`

Extend the bitarray from bytes, where each byte corresponds to a single
bit.  The byte `b'\x00'` maps to bit 0 and all other characters map to
bit 1.
This method, as well as the unpack method, are meant for efficient
transfer of data between bitarray objects to other python objects
(for example NumPy's ndarray object) which have a different memory view.


`pop(index=-1, /)` -> item

Return the i-th (default last) element and delete it from the bitarray.
Raises `IndexError` if bitarray is empty or index is out of range.


`remove(value, /)`

Remove the first occurrence of `bool(value)` in the bitarray.
Raises `ValueError` if item is not present.


`reverse()`

Reverse the order of bits in the array (in-place).


`search(bitarray, limit=<none>, /)` -> list

Searches for the given bitarray in self, and return the list of start
positions.
The optional argument limits the number of search results to the integer
specified.  By default, all search results are returned.


`setall(value, /)`

Set all bits in the bitarray to `bool(value)`.


`sort(reverse=False)`

Sort the bits in the array (in-place).


`to01()` -> str

Return a string containing '0's and '1's, representing the bits in the
bitarray object.


`tobytes()` -> bytes

Return the byte representation of the bitarray.
When the length of the bitarray is not a multiple of 8, the few remaining
bits (1..7) are considered to be 0.


`tofile(f, /)`

Write the byte representation of the bitarray to the file object f.
When the length of the bitarray is not a multiple of 8,
the remaining bits (1..7) are set to 0.


`tolist()` -> list

Return a list with the items (False or True) in the bitarray.
Note that the list object being created will require 32 or 64 times more
memory (depending on the machine architecture) than the bitarray object,
which may cause a memory error if the bitarray is very large.


`unpack(zero=b'\x00', one=b'\xff')` -> bytes

Return bytes containing one character for each bit in the bitarray,
using the specified mapping.


The frozenbitarray object:
--------------------------

`frozenbitarray(initializer=0, /, endian='big')` -> frozenbitarray

Return a frozenbitarray object, which is initialized the same way a bitarray
object is initialized.  A frozenbitarray is immutable and hashable.
Its contents cannot be altered after is created; however, it can be used as
a dictionary key.


Functions defined in the module:
--------------------------------

`test(verbosity=1, repeat=1)` -> TextTestResult

Run self-test, and return unittest.runner.TextTestResult object.


`bits2bytes(n, /)` -> int

Return the number of bytes necessary to store n bits.


`get_default_endian()` -> string

Return the default endianness for new bitarray objects being created.
Under normal circumstances, the return value is `big`.


Functions defined in bitarray.util:
-----------------------------------

`zeros(length, /, endian=None)` -> bitarray

Create a bitarray of length, with all values 0, and optional
endianness, which may be 'big', 'lillte'.


`make_endian(bitarray, endian, /)` -> bitarray

When the endianness of the given bitarray is different from `endian`,
return a new bitarray, with endianness `endian` and the same elements
as the original bitarray, i.e. even though the binary representation of the
new bitarray will be different, the returned bitarray will equal the original
one.
Otherwise (endianness is already `endian`) the original bitarray is returned
unchanged.


`rindex(bitarray, value=True, /)` -> int

Return the rightmost index of `bool(value)` in bitarray.
Raises `ValueError` if the value is not present.


`strip(bitarray, mode='right', /)` -> bitarray

Strip zeros from left, right or both ends.
Allowed values for mode are the strings: `left`, `right`, `both`


`count_n(a, n, /)` -> int

Find the smallest index `i` for which `a[:i].count() == n`.
Raises `ValueError`, when n exceeds the `a.count()`.


`count_and(a, b, /)` -> int

Returns `(a & b).count()`, but is more memory efficient,
as no intermediate bitarray object gets created.


`count_or(a, b, /)` -> int

Returns `(a | b).count()`, but is more memory efficient,
as no intermediate bitarray object gets created.


`count_xor(a, b, /)` -> int

Returns `(a ^ b).count()`, but is more memory efficient,
as no intermediate bitarray object gets created.


`subset(a, b, /)` -> bool

Return True if bitarray `a` is a subset of bitarray `b` (False otherwise).
`subset(a, b)` is equivalent to `(a & b).count() == a.count()` but is more
efficient since we can stop as soon as one mismatch is found, and no
intermediate bitarray object gets created.


`ba2hex(bitarray, /)` -> hexstr

Return a string containing with hexadecimal representation of
the bitarray (which has to be multiple of 4 in length).


`hex2ba(hexstr, /, endian=None)` -> bitarray

Bitarray of hexadecimal representation.
hexstr may contain any number of hex digits (upper or lower case).


`ba2int(bitarray, /)` -> int

Convert the given bitarray into an integer.
The bit-endianness of the bitarray is respected.


`int2ba(int, /, length=None, endian=None)` -> bitarray

Convert the given integer into a bitarray (with given endianness,
and no leading (big-endian) / trailing (little-endian) zeros).
If length is provided, the result will be of this length, and an
`OverflowError` will be raised, if the integer cannot be represented
within length bits.


`huffman_code(dict, /, endian=None)` -> dict

Given a frequency map, a dictionary mapping symbols to thier frequency,
calculate the Huffman code, i.e. a dict mapping those symbols to
bitarrays (with given endianness).  Note that the symbols may be any
hashable object (including `None`).


Change log
----------

2020-07-XX   1.4.1:

  * add official Python 3.9 support
  * improve many docstrings
  * C-level:
      - rewrote `.fromfile()` and `.tofile()` implementation,
        such that now the same code is used for Python 2 and 3.
        The new implementation is more memoery efficient on
        Python 3.
      - use memcmp() in richcompare to shortcut EQ/NE, when
        comparing two very large bitarrays for equality the
        speedup can easily be 100x
      - simplify how unpacking is handled
  * add more tests


*1.4.0* (2020-07-11):

  * add `.clear()` method (Python 3.3 added this method to lists)
  * avoid overallocation when bitarray objects are initially created
  * raise BufferError when resizing bitarrays which is exporting buffers
  * add example to study the resize() function
  * improve some error messages
  * add more tests
  * raise `NotImplementedError` with (useful message) when trying to call
    the `.fromstring()` or `.tostring()` methods, which have been removed
    in the last release


*1.3.0* (2020-07-06):

  * add `bitarray.util.make_endian()`
  * `util.ba2hex()` and `util.hex2ba()` now also support little-endian
  * add `bitarray.get_default_endian()`
  * made first argument of initializer a positional-only parameter
  * remove `.fromstring()` and `.tostring()` methods, these have been
    deprecated 8 years ago, since version 0.4.0
  * add `__all__` in `bitarray/__init__.py`
  * drop Python 3.3 and 3.4 support


*1.2.2* (2020-05-18):

  * `util.ba2hex` now always return a string object (instead of bytes object
    for Python 3), see issue #94
  * `util.hex2ba` allows a unicode object as input on Python 2
  * Determine 64-bitness of interpreter in a cross-platform fashion #91,
    in order to better support PyPy


Please find the complete change log
<a href="https://github.com/ilanschnell/bitarray/blob/master/CHANGE_LOG">here</a>.
