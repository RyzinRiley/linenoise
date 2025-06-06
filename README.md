# Linenoise

A minimal, zero-config, BSD licensed, readline replacement used in Redis,
MongoDB, Android and many other projects.

* Single and multi-line editing mode with the usual key bindings implemented.
* History handling.
* Completion.
* Hints (suggestions at the right of the prompt as you type).
* Multiplexing mode, with prompt hiding/restoring for asynchronous output.
* About ~850 lines (comments and spaces excluded) of BSD licensed source code.
* Only uses a subset of VT100 escapes (`ANSI.SYS` compatible).

## Can a line editing library be 20k lines of code?

Line editing with some support for history is a really important feature for command line utilities. Instead of retyping almost the same stuff again and again it's just much better to hit the up arrow and edit on syntax errors, or in order to try a slightly different command. But apparently code dealing with terminals is some sort of Black Magic: `readline` is 30k lines of code, `libedit` 20k. Is it reasonable to link small utilities to huge libraries just to get a minimal support for line editing?

So what usually happens is either:

 * Large programs with configure scripts disabling line editing if `readline` is not present in the system, or not supporting it at all since `readline` is GPL licensed and `libedit` (the BSD clone) is not as known and available as `readline` is (real world example of this problem: Tclsh).
 * Smaller programs not using a `configure` script not supporting line editing at all (A problem we had with `redis-cli`, for instance).
 
The result is a pollution of binaries without line editing support.

So I spent more or less two hours doing a reality check resulting in this little library: is it *really* needed for a line editing library to be 20k lines of code? Apparently not, it is possibe to get a very small, zero configuration, trivial to embed library, that solves the problem. Smaller programs will just include this, supporting line editing out of the box. Larger programs may use this little library or just checking with configure if readline/libedit is available and resorting to Linenoise if not.

## Terminals, in 2010.

Apparently almost every terminal you can happen to use today has some kind of support for basic VT100 escape sequences. So I tried to write a lib using just very basic VT100 features. The resulting library appears to work everywhere I tried to use it, and now can work even on `ANSI.SYS` compatible terminals, since no
VT220 specific sequences are used anymore.

The library is currently about 850 lines of code. In order to use it in your project just look at the `example.c` file in the source distribution, it is pretty straightforward. The library supports both a blocking mode and a multiplexing mode, see the API documentation later in this file for more information.

Linenoise is BSD-licensed code, so you can use both in free software and commercial software.

## Tested with...

 * Linux text only console ($TERM = linux)
 * Linux KDE terminal application ($TERM = xterm)
 * Linux xterm ($TERM = xterm)
 * Linux Buildroot ($TERM = vt100)
 * Mac OS X iTerm ($TERM = xterm)
 * Mac OS X default Terminal.app ($TERM = xterm)
 * OpenBSD 4.5 through an OSX Terminal.app ($TERM = screen)
 * IBM AIX 6.1
 * FreeBSD xterm ($TERM = xterm)
 * ANSI.SYS
 * Emacs comint mode ($TERM = dumb)

Please test it everywhere you can and report back!

## Let's push this forward!

Patches should be provided in the respect of Linenoise sensibility for small
easy to understand code.

Send feedbacks to antirez at gmail

# The API

Linenoise is very easy to use, and reading the example shipped with the
library should get you up to speed ASAP. Here is a list of API calls
and how to use them. Let's start with the simple blocking mode:
```cpp
char *linenoise(const char *prompt);
```
This is the main Linenoise call: it shows the user a prompt with line editing
and history capabilities. The prompt you specify is used as a prompt, that is,
it will be printed to the left of the cursor. The library returns a buffer
with the line composed by the user, or `NULL` on end of file or when there
is an out of memory condition.

When a tty is detected (the user is actually typing into a terminal session)
the maximum editable line length is `LINENOISE_MAX_LINE`. When instead the
standard input is not a tty, which happens every time you redirect a file
to a program, or use it in an Unix pipeline, there are no limits to the
length of the line that can be returned.

The returned line should be freed with the `free()` standard system call.
However sometimes it could happen that your program uses a different dynamic
allocation library, so you may also used `linenoiseFree` to make sure the
line is freed with the same allocator it was created.

The canonical loop used by a program using Linenoise will be something like
this:
```cpp
while((line = linenoise("hello> ")) != NULL) {
        printf("You wrote: %s\n", line);
        linenoiseFree(line); /* Or just free(line) if you use libc malloc. */
    }
```
## Single line VS multi line editing

By default, Linenoise uses single line editing, that is, a single row on the
screen will be used, and as the user types more, the text will scroll towards
left to make room. This works if your program is one where the user is
unlikely to write a lot of text, otherwise multi line editing, where multiple
screens rows are used, can be a lot more comfortable.

In order to enable multi line editing use the following API call:
```cpp
linenoiseSetMultiLine(1);
```
You can disable it using `0` as argument.

## History

Linenoise supporst history, so that the user does not have to retype
again and again the same things, but can use the down and up arrows in order
to search and re-edit already inserted lines of text.

The followings are the history API calls:
```cpp
int linenoiseHistoryAdd(const char *line);
int linenoiseHistorySetMaxLen(int len);
int linenoiseHistorySave(const char *filename);
int linenoiseHistoryLoad(const char *filename);
```

Use `linenoiseHistoryAdd` every time you want to add a new element
to the top of the history (it will be the first the user will see when
using the up arrow).

Note that for history to work, you have to set a length for the history
(which is zero by default, so history will be disabled if you don't set
a proper one). This is accomplished using the `linenoiseHistorySetMaxLen`
function.

Linenoise has direct support for persisting the history into an history
file. The functions `linenoiseHistorySave` and `linenoiseHistoryLoad` do
just that. Both functions return `-1` on error and `0` on success.

## Mask mode

Sometimes it is useful to allow the user to type passwords or other
secrets that should not be displayed. For such situations linenoise supports
a "mask mode" that will just replace the characters the user is typing 
with `*` characters, like in the following example:

    $ ./linenoise_example
    hello> get mykey
    echo: 'get mykey'
    hello> /mask
    hello> *********

You can enable and disable mask mode using the following two functions:

```cpp
void linenoiseMaskModeEnable(void);
void linenoiseMaskModeDisable(void);
```
## Completion

Linenoise supports completion, which is the ability to complete the user
input when she or he presses the `<TAB>` key.

In order to use completion, you need to register a completion callback, which
is called every time the user presses `<TAB>`. Your callback will return a
list of items that are completions for the current string.

The following is an example of registering a completion callback:

```cpp
linenoiseSetCompletionCallback(completion);
```

The completion must be a function returning `void` and getting as input
a `const char` pointer, which is the line the user has typed so far, and
a `linenoiseCompletions` object pointer, which is used as argument of
`linenoiseAddCompletion` in order to add completions inside the callback.
An example will make it more clear:

```cpp
void completion(const char *buf, linenoiseCompletions *lc) {
    if (buf[0] == 'h') {
            linenoiseAddCompletion(lc,"hello");
            linenoiseAddCompletion(lc,"hello there");
        }
    }
```

Basically in your completion callback, you inspect the input, and return
a list of items that are good completions by using `linenoiseAddCompletion`.

If you want to test the completion feature, compile the example program
with `make`, run it, type `h` and press `<TAB>`.

## Hints

Linenoise has a feature called *hints* which is very useful when you
use Linenoise in order to implement a REPL (Read Eval Print Loop) for
a program that accepts commands and arguments, but may also be useful in
other conditions.

The feature shows, on the right of the cursor, as the user types, hints that
may be useful. The hints can be displayed using a different color compared
to the color the user is typing, and can also be bold.

For example as the user starts to type `"git remote add"`, with hints it's
possible to show on the right of the prompt a string `<name> <url>`.

The feature works similarly to the history feature, using a callback.
To register the callback we use:
```c
linenoiseSetHintsCallback(hints);
```

The callback itself is implemented like this:

```c
char *hints(const char *buf, int *color, int *bold) {
    if (!strcasecmp(buf,"git remote add")) {
    *color = 35;
    *bold = 0;

return " <name> <url>";
     }
return NULL;
}
```

The callback function returns the string that should be displayed or `NULL`
if no hint is available for the text the user currently typed. The returned
string will be trimmed as needed depending on the number of columns available
on the screen.

It is possible to return a string allocated in dynamic way, by also registering
a function to deallocate the hint string once used:

```c
void linenoiseSetFreeHintsCallback(linenoiseFreeHintsCallback *);
```

The free hint callback will just receive the pointer and free the string
as needed (depending on how the hits callback allocated it).

As you can see in the example above, a `color` (in `xterm` color terminal codes)
can be provided together with a `bold` attribute. If no color is set, the
current terminal foreground color is used. If no bold attribute is set,
non-bold text is printed.

Color codes are:

    red = 31
    green = 32
    yellow = 33
    blue = 34
    magenta = 35
    cyan = 36
    white = 37;

## Screen handling

Sometimes you may want to clear the screen as a result of something the
user typed. You can do this by calling the following function:

```c
void linenoiseClearScreen(void);
```

## Asyncrhronous API

Sometimes you want to read from the keyboard but also from sockets or other
external events, and at the same time there could be input to display to the
user *while* the user is typing something. Let's call this the "IRC problem",
since if you want to write an IRC client with `linenoise`, without using
some fully featured `libcurses` approach, you will surely end having such an
issue.

Fortunately now a multiplexing friendly API exists, and it is just what the
blocking calls internally use. To start, we need to initialize a linenoise
context like this:

```c
struct linenoiseState ls;
char buf[1024];
    linenoiseEditStart(&ls,-1,-1,buf,sizeof(buf),"some prompt> ");
```

The two `-1` and `-1` arguments are the `stdin` / `stdout` descriptors. If they are
set to `-1`, linenoise will just use the default stdin/out file descriptors.
Now as soon as we have data from `stdin` (and we know it via `select(2)` or
some other way), we can ask linenoise to read the next character with:

```c
linenoiseEditFeed(&ls);
```

The function returns a `char` pointer: if the user didn't yet press enter
to provide a line to the program, it will return `linenoiseEditMore`, that
means we need to call `linenoiseEditFeed()` again when more data is
available. If the function returns non `NULL`, then this is a heap allocated
data (to be freed with `linenoiseFree()`) representing the user input.
When the function returns `NULL`, than the user pressed `CTRL-C` or `CTRL-D`
with an empty line, to quit the program, or there was some I/O error.

After each line is received (or if you want to quit the program, and exit raw mode), the following function needs to be called:

```c
linenoiseEditStop(&ls);
```

To start reading the next line, a new `linenoiseEditStart()` must
be called, in order to reset the state, and so forth, so a typical event
handler called when the standard input is readable, will work similarly
to the example below:

```c
void stdinHasSomeData(void) {
    char *line = linenoiseEditFeed(&LineNoiseState);
    if (line == linenoiseEditMore) return;
    linenoiseEditStop(&LineNoiseState);
    if (line == NULL) exit(0);

    printf("line: %s\n", line);
    linenoiseFree(line);
    linenoiseEditStart(&LineNoiseState,-1,-1,LineNoiseBuffer,sizeof(LineNoiseBuffer),"serial> ");
}
```

Now that we have a way to avoid blocking in the user input, we can use
two calls to hide/show the edited line, so that it is possible to also
show some input that we received (from sockets, Bluetooth, whatever) on
screen:

```c
linenoiseHide(&ls);
printf("some data...\n");
linenoiseShow(&ls);
```

To the API calls, the linenoise example C file implements a multiplexing
example using `select(2)` and the asynchronous API:

```c
struct linenoiseState ls;
char buf[1024];
    linenoiseEditStart(&ls,-1,-1,buf,sizeof(buf),"hello> ");

while(1) {
// Select(2) setup code removed...
     retval = select(ls.ifd+1, &readfds, NULL, NULL, &tv);
        if (retval == -1) {
            perror("select()");
     exit(1);
        } else if (retval) {
            line = linenoiseEditFeed(&ls);
/* A NULL return means: line editing is continuing.
* Otherwise the user hit enter or stopped editing
* (CTRL+C/D). */

    if (line != linenoiseEditMore) break;
        } else {
// Timeout occurred

static int counter = 0;
            linenoiseHide(&ls);
printf("Async output %d.\n", counter++);
            linenoiseShow(&ls);
        }
    }
linenoiseEditStop(&ls);
    if (line == NULL) exit(0); /* Ctrl+D/C. */
```

You can test the example by running the example program with the `--async` option.

## Related projects

* [Linenoise NG](https://github.com/arangodb/linenoise-ng) is a fork of Linenoise that aims to add more advanced features like UTF-8 support, Windows support and other features. Uses C++ instead of C as development language.
* [Linenoise-swift](https://github.com/andybest/linenoise-swift) is a reimplementation of Linenoise written in Swift.
