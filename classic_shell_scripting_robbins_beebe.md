# Classic Shell Scripting
Arnold Robbins & Nelson H. F. Beebe

# Chapter 2: Getting Started
## Self-Contained Scripts: The #! First Line
- `#!` causes the kernel to scan the rest of the line for the pathname of an interpreter to run the program.
  - Scans for a *single* option to be passed to the interpreter.
  - Avoid trailing whitespace after an option; it gets passed to the interpreter.
  - The bare option `-` says there are no more shell options.
    - Security feature to prevent spoofing attacks. E.g. `#! /bin/sh -`

## Basic Shell Constructs
### Commands and Arguments
- `--` signifies the end of options.
  - Options after `--` are treated as arguments.
- Semicolons separate commands on the same line, which are run in sequence.
  - Commands joined with `&` run concurrently.

### Variables
- Shell varaibles hold string values.
  - Can be empty. I.e. contain no characters.
  - No character limit.
<br><br>

---
## `echo`
*Usage*
`echo [string...]`

*Purpose*
* To produce output from shell scripts.

*Major options*
* None

*Behaviour*
* Prints each argument to stdout, separated by a single space and terminated by a newline.
* Interprets escape sequences.

*Caveats*
* Many versions support an `-n` option, which omits the final newline.
  * Not portable; not included by the POSIX standard.
---  
<br>

### Fancier Output with `printf`
* `printf format-string [arguments...]`
  * If there are more arguments than format specifications, `printf` cycles through the list.
* E.g. `printf "%s, %s!\n" Hello world`

### Basic I/O Redirection
* `Ctrl-D` indicates end of file.

**Rediretion and pipelines**
* Change standard input with `<`.
  * E.g. `tr -d '\r' < dos-file.txt ... `
* Change standard output with `>`.
  * E.g. `tr -d '\r' < dos-file.txt > unix-file.txt`
  * Creates the destination file if it doesn't exist.
  * Truncates existing files.
* Append to a file with `>>`.
  * Creates the destination file if it doesn't exist.
  * ```bash
    for f in dos-file*.txt
    do
      tr -d '\r' < $f >> big-unix-file.txt
    done
    ```
* Create pipelines with `|`.
* When constructing pipelines, try to reduce the amount of data at each stge for efficiency.
<br><br>

---
## `tr`
*Usage*
`tr [options] source-char-list replace-char-list`

*Purpose*
* To transliterate characters. E.g. converting upper to lowercase.
* Options allow removing characters and compressing runs of identical characters.


*Major options*
* `-c`: Complement. The characters that `tr` translates are those that are *not* in `source-char-list`. Usually used with `-d` or `-s`.
* `-C`: Like `-c` but works on multibyte characters, not binary byte values.
* `-d`: Delete characters in `source-char-list` instead of transliterating them.
* `-s`: Squeeze. Each sequence of repeated characters is replaced with a single instance of that character if it appears in `source-char-list`.

*Behaviour*
* Reads characters from standard input and writes them to standard output.
* Each input character in `source-char-list` is replaced with the corresponding character in `replace-char-list`.

*Caveats*
* `-C` operates on characters as specified by the current locale.
  * Not supported by all systems.
---
<br>

**Special files: `/dev/null` and `/dev/tty`**
* Data sent to `/dev/null` is thrown away, but the writing program believes it was successful.
* Reading from `/dev/null` returns EOF immediately.
* When `/dev/tty` is opened, Unix redirects it to the real terminal associated with the program.
  * Useful when reading input that must come from a human, such as passwords.
  * ```bash
      printf "Enter new password: " # Prompt for input
      stty -echo                    # Turn off echoing typed characters
      read pass < /dev/tty          # Read password
      stty echo                     # Turn echoing back on
    ```

### Basic Command Searching
* Empty components in `$PATH` mean "the current directory".
  * ```bash
      :/bin:/usr/bin      # Current directory first
      /bin:/usr/bin:      # Current directory last
      /bin::/usr/bin      # Current directory in the middle
    ```
  * Don't use the current directory in your path; it's a security problem.

## Accessing Shell Script Arguments
* Positional parameters represent a shell script's command-line arguments.
  * Within shell functions, they represent a function's arguments.
  * ```bash
      echo first arg is $1
      echo tenth arg is ${10}
    ```
* Example script to see if a user with a given name is logged in:
  * ```bash
      $ cat > finduser            # Create a new file
      #! /bin/sh

      # finduser --- see if user named by first argument is logged in

      who | grep $1
      ^D                          # End of file

      $ chmod +x finduser         # Make it executable

      $ mv finduser $HOME/bin     # Save it in our personal bin

      $ finduser betsy
    ```

## Simple Execution Tracing
* Execution tracing causes the shell to print each comand as it's executed, preceded by "+".
* Enable in a script with `set -x` to turn it on, and `set +x` to turn it off.
* ```bash
    $ cat > trace1.sh
    #! /bin/sh

    set -x                # Turn on tracing
    echo 1st echo         # Do something

    set +x                # Turn off tracing
    each 2nd echo         # Do something else
    ^D

    $ chmod +x trace1.sh

    $ ./trace1.sh
    + echo 1st echo
    1st echo
    + set +x
    2nd echo
  ```

## Internationalization and Localization
* Internationalization (i18n for short) is the process of designing software so that it can be adapted for specific user communities without having to change or recompile the code.
  * At minimum, this means wrapping strings in library calls that handle runtime lookup of translations in message catalogs.
* Localization is the process of adapting internationalized software for use by specific user communities.
  * May require translating documentation, text output, currency, dates, times, etc.

# Chapter 3: Searching and Substitutions
## Searching for Text

---
## `grep`
*Usage*
`grep [options...] pattern-spec [files...]`

*Purpose*
* To print lines of text that match one or more patterns, often as the first stage in a pipeline.

*Major options*
* `-E`: Extended. Match using extended regular expressions.
* `-F`: Match using fixed strings.
* `-e pat-list`: Specifies that its argument is a pattern, even if it starts with a minus sign like an option.
* `-f pat-file`: File. Read patterns from pat-file.
* `-i`: Ignore. Ignore case.
* `-l`: List. List the names of files that match the pattern instead of the lines within the files.
* `-q`: Quiet. `grep` writes no lines to stdout, but exits successfully if it matches the pattern and unsuccessfully otherwise.
* `-s`: Suppress error messages. Often used with `-q`.
* `-v`: Invert. Print lines that don't match the pattern.

*Behaviour*
* Reads through each file named on the command line. When a line matches the pattern, print the line.
* When multiple files are named, `grep` prceedes each line with the filename and a colon.
* Uses Basic Regular Expressions (BREs) by default.

*Caveats*
* You can use multiple `-e` and `-f` options to build up a list of patterns to search for.
---
<br>

### Simple grep
* `who | grep -F austen`
  * `-F` is used to search for fixed strings.
    * If pattern doesn't contain regex metacharacters, `grep`'s default behaviour is the same as `-F`.

## Regular Expressions
<table>
  <thead>
    <tr>
      <th><strong>Character</strong></th>
      <th><strong>BRE/ERE</strong></th>
      <th><strong>Meaning</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>\</td>
      <td>Both</td>
      <td>Usually escapes a special character. In BREs, enables special characters such as \(...\).</td>
    </tr>
    <tr>
      <td>.</td>
      <td>Both</td>
      <td>Match any single character except NUL.</td>
    </tr>
    <tr>
      <td>*</td>
      <td>Both</td>
      <td>Match zero or more of the preceding character.</td>
    </tr>
    <tr>
      <td>^</td>
      <td>Both</td>
      <td>Match at the beginning of a line or string. BRE: Special only at the beginning of a regex. ERE: Special everywhere.</td>
    </tr>
    <tr>
      <td>$</td>
      <td>Both</td>
      <td>Match at the end of a line or string. BRE: Special only at the beginning of a regex. ERE: Special everywhere.</td>
    </tr>
    <tr>
      <td>[...]</td>
      <td>Both</td>
      <td>Bracket expression. Matches <strong>one</strong> of the enclosed characters. ^ as the first character negates the list. - indicates a range of consecutive characters. - or ] as the first character are counted as members of the list. Metacharacters are treated as literal members of the list.</td>
    </tr>
    <tr>
      <td>\{n,m\}</td>
      <td>BRE</td>
      <td>Interval expression, matching a range of occurrences of the preceding character.</td>
    </tr>
    <tr>
      <td>\( \)</td>
      <td>BRE</td>
      <td>Save the pattern enclosed between the parentheses in a special holding space. The text matched by the pattern can be reused later in the same pattern by the escape sequences \1 to \9.</td>
    </tr>
    <tr>
      <td>{n,m}</td>
      <td>ERE</td>
      <td>Like the BRE interval expression, but does not require escaping.</td>
    </tr>
    <tr>
      <td>+</td>
      <td>ERE</td>
      <td>Match one or more instances of the preceding character.</td>
    </tr>
    <tr>
      <td>?</td>
      <td>ERE</td>
      <td>Match zero or one instances of the preceding character.</td>
    </tr>
    <tr>
      <td>|</td>
      <td>ERE</td>
      <td>Match the regular expression specified before or after.</td>
    </tr>
    <tr>
      <td>( )</td>
      <td>ERE</td>
      <td>Apply a match to the encolsed group of regular expressions.</td>
    </tr>
  </tbody>
</table>