This is a fork from Ned Batchelder's Cog, a python text file generation tool which lets you use pieces of Python code as generators in your source files to generate whatever text you need.

The original repo can be found on bitbucket here

http://bitbucket.org/ned/cog

and the tutorial blog post from the original autor can be found here:

https://nedbatchelder.com/code/cog/

which contents I'm posting here just for brevity:

========================================

# Cog

Cog is a file generation tool. It lets you use pieces of Python code as generators in your source files to generate whatever text you need.

The sections below are:

* What does it do?
* Design
* Installation
* Writing the source files
* The cog module
* Running cog
* History
* Feedback
* See Also

What does it do?
------
Cog transforms files in a very simple way: it finds chunks of Python code embedded in them, executes the Python code, and inserts its output back into the original file. The file can contain whatever text you like around the Python code. It will usually be source code.

For example, if you run this file through cog:

```
// This is my C++ file.
...
/*[[[cog
import cog
fnames = ['DoSomething', 'DoAnotherThing', 'DoLastThing']
for fn in fnames:
    cog.outl("void %s();" % fn)
]]]*/
//[[[end]]]
...
```

it will come out like this:

```
// This is my C++ file.
...
/*[[[cog
import cog
fnames = ['DoSomething', 'DoAnotherThing', 'DoLastThing']
for fn in fnames:
    cog.outl("void %s();" % fn)
]]]*/
void DoSomething();
void DoAnotherThing();
void DoLastThing();
//[[[end]]]
...
```

Lines with triple square brackets are marker lines. The lines between [[[cog and ]]] are the generator Python code. The lines between ]]] and [[[end]]] are the output from the generator.

When cog runs, it discards the last generated Python output, executes the generator Python code, and writes its generated output into the file. All text lines outside of the special markers are passed through unchanged.

The cog marker lines can contain any text in addition to the triple square bracket tokens. This makes it possible to hide the generator Python code from the source file. In the sample above, the entire chunk of Python code is a C++ comment, so the Python code can be left in place while the file is treated as C++ code.

Design
------
Cog is designed to be easy to run. It writes its results back into the original file while retaining the code it executed. This means cog can be run any number of times on the same file. Rather than have a source generator file, and a separate output file, typically cog is run with one file serving as both generator and output.

Because the marker lines accommodate any language syntax, the markers can hide the cog Python code from the source file. This means cog files can be checked into source control without worrying about keeping the source files separate from the output files, without modifying build procedures, and so on.

I experimented with using a templating engine for generating code, and found myself constantly struggling with white space in the generated output, and mentally converting from the Python code I could imagine, into its templating equivalent. The advantages of a templating system (that most of the code could be entered literally) were lost as the code generation tasks became more complex, and the generation process needed more logic.

Cog lets you use the full power of Python for text generation, without a templating system dumbing down your tools for you.

Installation
------
Cog requires Python 2.6, 2.7, 3.3, 3.4, 3.5, 3.6, or Jython 2.5.

Cog is installed with a standard Python distutils script:

Download Cog from the Python Package Index.
Unpack the distribution archive somewhere.
Run the setup.py script that was unpacked:
```
$ python setup.py install
```
You should now have cog.py in your Python scripts directory.

License
------
Cog is distributed under the MIT license. Use it to spread goodness through the world.

Writing the source files
------
Source files to be run through cog are mostly just plain text that will be passed through untouched. The Python code in your source file is standard Python code. Any way you want to use Python to generate text to go into your file is fine. Each chunk of Python code (between the [[[cog and ]]] lines) is called a generator and is executed in sequence.

The output area for each generator (between the ]]] and [[[end]]] lines) is deleted, and the output of running the Python code is inserted in its place. To accommodate all source file types, the format of the marker lines is irrelevant. If the line contains the special character sequence, the whole line is taken as a marker. Any of these lines mark the beginning of executable Python code:

```
//[[[cog
/* cog starts now: [[[cog */
-- [[[cog (this is cog Python code)
#if 0 // [[[cog
```

Cog can also be used in languages without multi-line comments. If the marker lines all have the same text before the triple brackets, and all the lines in the generator code also have this text as a prefix, then the prefixes are removed from all the generator lines before execution. For example, in a SQL file, this:

```
--[[[cog
--   import cog
--   for table in ['customers', 'orders', 'suppliers']:
--      cog.outl("drop table %s;" % table)
--]]]
--[[[end]]]
```

will produce this:

```sql
--[[[cog
--   import cog
--   for table in ['customers', 'orders', 'suppliers']:
--      cog.outl("drop table %s;" % table)
--]]]
drop table customers;
drop table orders;
drop table suppliers;
--[[[end]]]
```

Finally, a compact form can be used for single-line generators. The begin-code marker and the end-code marker can appear on the same line, and all the text between them will be taken as a single Python line:

```
// blah blah
//[[[cog import MyModule as m; m.generateCode() ]]]
//[[[end]]]
```

You can also use this form to simply import a module. The top-level statements in the module can generate the code.

If you have special requirements for the syntax of your file, you can use the --markers option to define new markers.

If there are multiple generators in the same file, they are executed with the same globals dictionary, so it is as if they were all one Python module.

Cog tries to do the right thing with white space. Your Python code can be block-indented to match the surrounding text in the source file, and cog will re-indent the output to fit as well. All of the output for a generator is collected as a block of text, a common whitespace prefix is removed, and then the block is indented to match the indentation of the cog generator. This means the left-most non-whitespace character in your output will have the same indentation as the begin-code marker line. Other lines in your output keep their relative indentation.

The cog module
------
A module called cog provides the functions you call to produce output into your file. The functions are:

cog.out(sOut=’’ [, dedent=False][, trimblanklines=False])
Writes text to the output.
sOut is the string to write to the output.
If dedent is True, then common initial white space is removed from the lines in sOut before adding them to the output. If trimblanklines is True, then an initial and trailing blank line are removed from sOut before adding them to the output. Together, these option arguments make it easier to use multi-line strings, and they only are useful for multi-line strings:

```
cog.out("""
    These are lines I
    want to write into my source file.
""", dedent=True, trimblanklines=True)
```

cog.outl
Same as cog.out, but adds a trailing newline.
cog.msg(msg)
Prints msg to stdout with a “Message: ” prefix.
cog.error(msg)
Raises an exception with msg as the text. No traceback is included, so that non-Python programmers using your code generators won’t be scared.
cog.inFile
An attribute, the path of the input file.
cog.outFile
An attribute, the path of the output file.
cog.firstLineNum
An attribute, the line number of the first line of Python code in the generator. This can be used to distinguish between two generators in the same input file, if needed.
cog.previous
An attribute, the text output of the previous run of this generator. This can be used for whatever purpose you like, including outputting again with cog.out().

Running cog
------
Cog is a command-line utility which takes arguments in standard form.

```
cog - generate code with inlined Python code.

cog [OPTIONS] [INFILE | @FILELIST] ...

INFILE is the name of an input file, '-' will read from stdin.
FILELIST is the name of a text file containing file names or
    other @FILELISTs.

OPTIONS:
    -c          Checksum the output to protect it against accidental change.
    -d          Delete the generator code from the output file.
    -D name=val Define a global string available to your generator code.
    -e          Warn if a file has no cog code in it.
    -I PATH     Add PATH to the list of directories for data files and modules.
    -n ENCODING Use ENCODING when reading and writing files.
    -o OUTNAME  Write the output to OUTNAME.
    -r          Replace the input file with the output.
    -s STRING   Suffix all generated output lines with STRING.
    -U          Write the output with Unix newlines (only LF line-endings).
    -w CMD      Use CMD if the output file needs to be made writable.
                    A %s in the CMD will be filled with the filename.
    -x          Excise all the generated output without running the generators.
    -z          The end-output marker can be omitted, and is assumed at eof.
    -v          Print the version of cog and exit.
    --verbosity=VERBOSITY
                Control the amount of output. 2 (the default) lists all files,
                1 lists only changed files, 0 lists no files.
    --markers='START END END-OUTPUT'
                The patterns surrounding cog inline instructions. Should
                include three values separated by spaces, the start, end,
                and end-output markers. Defaults to '[[[cog ]]] [[[end]]]'.
    -h          Print this help.
```

In addition to running cog as a command on the command line:
```
$ cog [options] [arguments]
```
you can also invoke it as a module with the Python interpreter:
```
$ python -m cogapp [options] [arguments]
```
Note that the Python module is called “cogapp”.

Input files
------
Files on the command line are processed as input files. All input files are assumed to be UTF-8 encoded. Using a minus for a filename (-) will read the standard input.

Files can also be listed in a text file named on the command line with an @:
```
$ cog @files_to_cog.txt
```
These @-files can be nested, and each line can contain switches as well as a file to process. For example, you can create a file cogfiles.txt:

```
cogfiles.txt

# These are the files I run through cog
mycode.cpp
myothercode.cpp
myschema.sql -s " --**cogged**"
readme.txt -s ""
```
then invoke cog like this:
```
cog -s " //**cogged**" @cogfiles.txt
```
Now cog will process four files, using C++ syntax for markers on all the C++ files, SQL syntax for the .sql file, and no markers at all on the readme.txt file.

As another example, cogfiles2.txt could be:
```
cogfiles2.txt

template.h -D thefile=data1.xml -o data1.h
template.h -D thefile=data2.xml -o data2.h
```
with cog invoked like this:
```
cog -D version=3.4.1 @cogfiles2.txt
```
Cog will process template.h twice, creating both data1.h and data2.h. Both executions would define the variable version as “3.4.1”, but the first run would have thefile equal to “data1.xml” and the second run would have thefile equal to “data2.xml”.

Overwriting files
------
The -r flag tells cog to write the output back to the input file. If the input file is not writable (for example, because it has not been checked out of a source control system), a command to make the file writable can be provided with -w:
```
$ cog -r -w "p4 edit %s" @files_to_cog.txt
```
Setting globals
------
Global values can be set from the command line with the -D flag. For example, invoking Cog like this:
```
cog -D thefile=fooey.xml mycode.txt
```
will run Cog over mycode.txt, but first define a global variable called thefile with a value of “fooey.xml”. This variable can then be referenced in your generator code. You can provide multiple -D arguments on the command line, and all will be defined and available.

The value is always interpreted as a Python string, to simplify the problem of quoting. This means that:
```
cog -D NUM_TO_DO=12
```
will define NUM_TO_DO not as the integer 12, but as the string “12”, which are different and not equal values in Python. Use int(NUM_TO_DO) to get the numeric value.

Checksummed output
------
If cog is run with the -c flag, then generated output is accompanied by a checksum:

```
--[[[cog
--   import cog
--   for i in range(10):
--      cog.out("%d " % i)
--]]]
0 1 2 3 4 5 6 7 8 9
--[[[end]]] (checksum: bd7715304529f66c4d3493e786bb0f1f)
```

If the generated code is edited by a misguided developer, the next time cog is run, the checksum won’t match, and cog will stop to avoid overwriting the edited code.

Output line suffixes
------
To make it easier to identify generated lines when grepping your source files, the -s switch provides a suffix which is appended to every non-blank text line generated by Cog. For example, with this input file (mycode.txt):

mycode.txt

```
[[[cog
cog.outl('Three times:\n')
for i in range(3):
    cog.outl('This is line %d' % i)
]]]
[[[end]]]
```

invoking cog like this:
```
cog -s " //(generated)" mycode.txt
```
will produce this output:

```
[[[cog
cog.outl('Three times:\n')
for i in range(3):
    cog.outl('This is line %d' % i)
]]]
Three times: //(generated)

This is line 0 //(generated)
This is line 1 //(generated)
This is line 2 //(generated)
[[[end]]]
```

Miscellaneous
------
The -n option lets you tell cog what encoding to use when reading and writing files.

The --verbose option lets you control how much cog should chatter about the files it is cogging. --verbose=2 is the default: cog will name every file it considers, and whether it has changed. --verbose=1 will only name the changed files. --verbose=0 won’t mention any files at all.

The --markers option lets you control the syntax of the marker lines. The value must be a string with two spaces in it. The three markers are the three pieces separated by the spaces. The default value for markers is “[[cog ]]] [[[end]]]”.

The -x flag tells cog to delete the old generated output without running the generators. This lets you remove all the generated output from a source file.

The -d flag tells cog to delete the generators from the output file. This lets you generate code in a public file but not have to show the generator to your customers.

The -U flag causes the output file to use pure Unix newlines rather than the platform’s native line endings. You can use this on Windows to produce Unix-style output files.

The -I flag adds a directory to the path used to find Python modules.

The -z flag lets you omit the [[[end]]] marker line, and it will be assumed at the end of the file.

History
------
Cog’s change log is on a separate change page.

These are changes to Cog over time.

Version 2.5.1, October 19 2016
------
Corrected a long-standing oversight: added a LICENSE.txt file.

Version 2.5, February 13 2016
------
When specifying an output file with -o, directories will be created as needed to write the file. Thanks, Jean-François Giraud.

Version 2.4, January 11 2015
------
A --markers option lets you control the three markers that separate the cog code and result from the rest of the file. Thanks, Doug Hellmann.
A -n=ENCODING option that lets you specify the encoding for the input and output files. Thanks, Petr Gladkiy.
A --verbose option that lets you control how much chatter is in the output while cogging.

Version 2.3, February 27 2012
------
Python 3 is now supported. Older Pythons (2.5 and below) no longer are.
Added the cog.previous attribute to get the output from the last time cog was run.
An input file name of “-” will read input from standard in.
Cog can now be run with “python -m cogapp [args]”.
All files are assumed to be encoded with UTF-8.

Version 2.2, June 25 2009
------
Jython 2.5 is now supported.
Removed a warning about using the no-longer-recommended md5 module.
Removed handyxml: most Cog users don’t need it.

Version 2.1, May 22 2008
------
Added the -U switch to create Unix newlines on Windows.
Improved argument validation: -d can be used with stdout-destined output, and switches are validated for every line of an @file, to prevent bad interactions.

Version 2.0, October 6 2005
------
Incompatible changes:

Python 2.2 is no longer supported.
In 1.4, you could put some generator code on the [[[cog line and some on the ]]] line, to make the generators more compact. Unfortunately, this also made it more difficult to seamlessly embed those markers in source files of all kinds. Now code is only allowed on marker lines when the entire generator is single-line.
In 1.x, you could leave out the [[[end]]] marker, and it would be assumed at the end of the file. Now that behavior must be enabled with a -z switch. Without the switch, omitting the end marker is an error.
Beneficial changes:

The new -d switch removes all the generator code from the output file while running it to generate output (thanks, Blake).
The new -D switch lets you define global string values for the generators.
The new -s switch lets you mark generated output lines with a suffix.
@-files now can have command line switches in addition to file names.
Cog error messages now print without a traceback, and use a format similar to compiler error messages, so that clicking the message will likely bring you to the spot in your code (thanks, Mike).
New cog method #1: cog.error(msg) will raise an error and end processing without creating a scary Python traceback (thanks, Alexander).
New cog method #2: cog.msg(msg) will print the msg to stdout. This is better than print because it allows for all cog output to be managed through Cog.
The sequence of Cog marker lines is much stricter. This helps to ensure that Cog isn’t eating up your precious source code (thanks, Kevin).

Version 1.4, February 25 2005
------
Added the -x switch to excise generated output.
Added the -c switch to checksum the generated output.

Version 1.3, December 30 2004
------
All of the generators in a single file are now run with a common globals dictionary, so that state may be carried from one to the next.

Version 1.2, December 29 2004
------
Added module attributes cog.inFile, cog.outFile, and cog.firstLineNum.
Made the sOut argument optional in cog.out and cog.outl.
Added the compact one-line form of cog markers.
Some warning messages weren’t properly printing the file name.

Version 1.12, June 21 2004
------
Changed all the line endings in the source to the more-portable LF from the Windows-only CRLF.

Version 1.11, June 5 2004
------
Just bug fixes:

Cog’s whitespace handling deals correctly with a completely blank line (no whitespace at all) in a chunk of Cog code.
Elements returned by handyxml can now have attributes assigned to them after parsing.

Version 1.1, March 21 2004
------
Now if the cog marker lines and all the lines they contain have the same prefix characters, then the prefix is removed from each line. This allows cog to be used with languages that don’t support multi-line comments.
Ensure the last line of the output ends with a newline, or it will merge with the end marker, ruining cog’s idempotency.
Add the -v command line option, to print the version.
Running cog with no options prints the usage help.

Version 1.0, February 10 2004
------
First version.

Feedback
------
I’d love to hear about your successes or difficulties using cog. Comment here, or send me a note.

See Also
------
There are a handful of other implementations of the ideas in Cog:
* [Argent](http://rubyforge.org/projects/argent/) is a Ruby implementation.
* [Precog](http://babylon.idlevice.co.uk/php/precog/) is a PHP implementation.
* [PCG](http://www.crazilocks.com/pcg.html) is a Perl implementation.
* [Templarian](http://sourceforge.net/projects/templarian) is a similar tool, also in Python.
* [Nocog](http://www.larryhastings.com/programming/nocog/) is a build tool to detect files that should be run through cog.
You might like to read:
* [Cog: A Code Generation Tool Written in Python](http://www.python.org/about/success/cog/), the Python Success Story I wrote about Cog.
* [My blog](https://nedbatchelder.com/blog), where I ramble on about software and other things that interest me.