# Docs: Staging

Inside a VM assigned to a user, a "conductor" program (`/rto/run`) is responsible for orchestrating containers holding running user code. The conductor is responsible for running interpreters, compilers, and compiled code within these containers, as well as inserting files, streaming input/output, and communicating with the server outside the VM. The conductor interacts with the container according to instructions specified in a configuration file, and its actions occur in a predictable order. This is "staging".

As an example, a simple Python configuration would consist of two stages:

1. Insert a file, `code.py`, containing the user's code
2. Run `python` inside the container, with user-specified options, the source file `code.py`, and user-provided command-line arguments, while streaming standard I/O to the outside world

A third stage could add debug information, like `python`'s return code, the time the program took to run, and whether or not the code terminated due to a timeout.

In contrast, a compiled language like C needs a minimum of three stages:

1. Insert a file, `code.c`, containing the user's code
2. Run `gcc` inside the container, with user-specified compilation options and the `code.c` source file
3. Run the resulting executable, with user-provided command-line arguments, while streaming standard I/O to the outside world

## (Advanced) Script-managed staging

Since the compiler needs to be run before the code itself can be, an additional stage is required. It would be possible to do this in a single stage, by including a script in the container's root filesystem that itself sequentiually handles compiling then running the code:

1. Insert a file, `code.c`, containing the user's code
2. Run `run.sh` inside the container, with user-specified options and user-provided command-line arguments, while streaming standard I/O to the outside world

This approach could be used to support languages which must be run in ways more advanced than can be described using the conductor's staging configuration rules. In order to properly support live-streaming of I/O, and minimize overhead, this should preferably be done with a compiled language, with careful attention paid to standard I/O handling. A template of such a program in Rust will be included in this GitHub organization at some point.

## Staging at a low level

A `runc` container's lifetime starts with a `create` command, which sets up its environment, but does not run any processes within it. Processes may be started in the container using either a `start` command, or an `exec` command. A `start` runs an initial process, and stops the container once this process finishes. Since this behavior would be undesirable with staging which runs multiple processes sequentially, only `exec`, which starts a process at any time, is used.

## Staging directives

Staging is specified in JSON. The `staging` property of a language configuration may be an array, or as a shorthand, a single object which will behave identically to one wrapped in a single-item array. Arrays of directives are followed in order, and if a directive fails, no further directives run. Directives are formatted as objects, with a `directive` property specifying the type. Directive types include:

- `readFile`: Reads a file (`file`) to a destination stream (`dst`), optionally closing it (`close`), and failing if the file doesn't exist (must run within a `spawnContainer` directive)
- `writeFile`: Writes to a file (`file`), optionally in append mode (`append`), from a source string or stream (`src`), and failing if the file's directory is missing its existence doesn't match optional expectations (`exists`) (must run within a `spawnContainer` directive)
- `run`: Runs an executable (`run`), with parameters including `args`, `stdin`/`stdout`/`stderr`, `tty`, `env`, and optional constant paramters like `uid`, `gid`, and `umask`, failing if its return code isn't `0` (unless `ignoreCode`, `successCodes`, or `failCodes` is set) (must run within a `spawnContainer` directive)
- `simul`: Runs multiple directives (`directives`) simultaneously, finishing when all directives finish
- `closeStream`: Closes a stream (`stream`)
- `waitStream`: Blocks until a stream (`stream`) closes
- `conditional`: Runs directives (`directives`) if a condition (`condition`) is truthy, and optionally a second set of directives otherwise (`otherwise`)
- `forkCasesSimul`: Runs directives (`directives`) for each case (no-op in single-case or terminal mode), concurrently (cannot run after, simultaneously with, or within another `forkCases*` directive)
- `forkCasesSeq`: Runs directives (`directives`) for each case (no-op in single-case or terminal mode), sequentially (cannot run after, simultaneously with, or within another `forkCases*` directive)
- `groupCasesSqrt`: Runs directives (`directives`) simultaneously with a subset of cases, the square root of the cases in scope (cannot run within, and must contain, a `forkCases*` directive)
- `groupCasesOf`: Runs directives (`directives`) simultaneously with a subset of cases, with each group being a certain size (`size`) (cannot run within, and must contain, a `forkCases*` directive)
- `groupCases`: Runs directives (`directives`) simultaneously with a subset of cases, with N groups (`groups`) (cannot run within, and must contain, a `forkCases*` directive)
- `spawnContainer`: Spawns a container to runs directives (`directives`) in, which all `readFile`, `writeFile`, and `run` directives will use (cannot run within another `spawnContainer` directive), with optional constant parameters including `mounts`, `namespaces`, and `capabilties`

Different `type`s of conditions (if no arguments provided, condition can be a `type` string instead of an object):

- `not`: Inverts a condition (`condition`)
- `and`: Truthy if a list of conditions (`conditions`) are all truthy
- `or`: Truthy if any condition in a list of conditions (`conditions`) is truthy
- `streamClosed`: Truthy if a stream (`stream`) is closed
- `ttyMode`: Truthy if running in terminal mode
- `nonTtyMode`: Truthy if running in a mode other than terminal
- `singleCaseMode`: Truthy if running in single-case mode
- `multiCaseMode`: Truthy if running in multi-case mode
- `caseCountIs`: Truthy if the case count in multi-case mode matches a certain int (`count`)
- `caseCountLess`: Truthy if the case count in multi-case mode is less than a certain int (`count`)
- `caseCountGreater`: Truthy if the case count in multi-case mode is greater than a certain int (`count`)
- `codeSuccessful`: Truthy if the return code of the last `run` directive (or `0` if none) is `0`
- `codeIs`: Truthy if the return code of the last run directive (or `0` if none) is a certain code (`code`)
- `codeIsIn`: Truthy if the return code of the last run directive (or `0` if none) is in a specified list (`codeList`) or range (`codeBounds`)
- `envExists`: Truthy if a variable (`var`) exists in an environment (`env`)
- `envIs`: Truthy if a variable (`var`) is a certain value (`value`) in an environment (`env`)
- `envLess`: Truthy if a variable (`var`) is less than a certain numeric or lexicographic string value (`value`) in an environment (`env`)
- `envGreater`: Truthy if a variable (`var`) is greater than a certain numeric or lexicographic string value (`value`) in an environment (`env`)
- `containsArgument`: Truthy if an arguments list (`args`) contains an argument (`arg`)
- `optionHasArg`: Truthy if an option (`option`) is in an options list (`options`) and has an argument (according to an optional set of `parsingRules`)
- `optionIs`: Truthy if an option (`option`) is in an options list (`options`) and has an argument which is a certain value (`value`) (according to an optional set of `parsingRules`)
- `optionLess`: Truthy if an option (`option`) is in an options list (`options`) and has an argument which is less than a certain numeric or lexicographic value (`value`) (according to an optional set of `parsingRules`)
- `optionGreater`: Truthy if an option (`option`) is in an options list (`options`) and has an argument which is greater than a certain numeric or lexicographic value (`value`) (according to an optional set of `parsingRules`)
- `orphanOption`: Truthy if an argument in an options list (`options`) has no associated flag (according to an optional set of `parsingRules`), which could cause conflict with a file name
- `inputContains`: Truthy if an input string or stream (`output`) contains a certain string (`string`)
- `outputContains`: Truthy if an output string or stream (`output`) contains a certain string (`string`)
- `empty`: Truthy if an args/options list, input/output string, or stream is empty
- `sizeIs`: Truthy if the size of an args/options list, input/output string, or stream is a certain int (`size`)
- `sizeLess`: Truthy if the size of an args/options list, input/output string, or stream is less than a certain int (`size`)
- `sizeGreater`: Truthy if the size of an args/options list, input/output string, or stream is greater than a certain int (`size`)
- `finalInSimul`: Truthy if the directive is the final running in the deepest nested `simul` block
- `fileExists`: Truthy if a file (`file`) exists
- `dirExists`: Truthy if a directory (`dir`) exists

## Modes

The conductor can run an instance in three modes:

- Single-case
- Terminal
- Multi-case

In the default single-case mode, `tty` is false for any `run` directives unless otherwise specified, input is a string or closed stream, and the directives are run once, in order, in a single container. In terminal mode, `tty` is true for any `run` directives unless otherwise specified, input may be an open stream, and the directives are run once, in order, in a single container. In multi-case mode, `tty` is false for any `run` directives unless otherwise specified, input is a string or closed stream, and the directives are run once for each of a number of cases, in a single container or separate containers, concurrently or sequentially. Cases may differ in input, arguments, options, or multiple of the above.

## Data types

Data received from the client can be any of the following types:

- String (byte string of a variable but finite length)
- Stream (byte string transmitted incrementally, only possible in terminal mode)
- Args/options array (array of byte strings)
- Environment (map of environment variables)

Data sent to the client can only be streams, in any mode.

### Files

- **`read-file`:** Reads a file
  - `file` (string): The absolute path of the file
  - `dest` (object): A `string-dest` to write the file's contents to
- **`write-file`:** Writes to a file
  - `file` (string): The absolute path of the file
  - `source` (object): A `string-source` to write to the file
  - `append` (OPTIONAL; bool = false): Whether or not to append to the file
- **`rm-file`:** Removes a file
  - `file` (string): The absolute path of the file
- **`rm-dir`:** Removes a directory
  - `dir` (string): The absolute path of the directory
  - `conts` (OPTIONAL; bool = false): Whether or not to delete the directory's contents (if `false` and the directory is non-empty, will fail)
- **`ch-file`:** Changes the owner, group, and/or permissions of a file or directory
  - `file` (string): The absolute path of the file
  - `owner` (OPTIONAL; string or number): The user or UID to change the file's owner to (no change if not included)
  - `group` (OPTIONAL; string or number): The group or GID to change the file's group tp (no change if not included)
  - `mode` (OPTIONAL; number): The mode to change the file's permissions to (cannot be used with `mode-0-mask` or `mode-1-mask`)
  - `mode-0-mask` (OPTIONAL; number): A mask to bit clear the mode of the file (cannot clear a bit also set by `mode-1-mask`)
  - `mode-1-mask` (OPTIONAL; number): A mask to bit set the mode of the file (cannot set a bit also cleared by `mode-0-mask`)
- **`list-dir`:** Lists the contents of a directory
  - `dir` (string): The absolute path of the directory
  - `dest` (object): An `array-dest` to write the directory's contents to
- **`move-file`:** Moves a file or directory
  - `src-file` (string): The absolute path of the file or directory to move
  - `dst-file` (string): The absolute path of the destination
  - possibly a `-m`
- **`copy-file`:** Copies a file or directory
  - `src-file` (string): The absolute path of the file or directory to copy
  - `dst-file` (string): The absolute path of the destination

### Running

- **`run`:** Runs an executable
  - `run` (string): The absolute path of the executable
  - `args` (OPTIONAL; object): An `array-source` of arguments (defaults to no arguments)
  - `env` (OPTIONAL; object): A `dict-source` of environment variables (defaults to no environment variables)
  - `stdin` (OPTIONAL; object or `null`): A `string-source` to pass as STDIN (`null` is no STDIN)
  - `stdout` (OPTIONAL; object or `null`): A `string-dest` to write STDOUT to (`null` is ignore STDOUT)
  - `stderr` (OPTIONAL; object or `null`): A `string-dest` to write STDERR to (`null` is ignore STDERR)
  - `tty` (OPTIONAL; bool = `false`): Whether or not to allocate a pseudo-TTY
  - `user` (OPTIONAL; string or number): The user or UID to run the process as
  - `group` (OPTIONAL; string or number): The group or GID to run the process as
  - `umask` (OPTIONAL; number): The `umask` to give the user
  - `additional-groups` (OPTIONAL; array of strings or numbers): Additional groups or GIDs to add to the process
  - console size? capabilities? rlimits?
- **`kill`:** Kills a running process inside the container
  - ???

## Timing and control flow

- **`simul`:** Runs multiple directives simultaneously, finishing when all directives finish
- **`try-simul`:** Runs multiple directives simultaneously, finishing when all directives finish successfully or any directive fails
- **`first-simul`:** Runs multiple directives simultaneously, finishing when the first directive finishes
- **`try-first-simul`:** Runs multiple directives simulatenously, finishing when the first directive succeeds
- **`ignore-fails`:** Runs a sequence of directives, continuing if one fails
- **`alternatives`:** Runs a sequence of directives, until one succeeds
- **`wait`:** Waits before running a sequence of directives
- **`conditional`:** Runs a directive (or sequence of directives), if a condition is true

## Data types

Staging works with the following data types:

- String: Byte string containing (code)
- Array: Array of byte strings (arguments, command line options)
- Option: Tagged union (settings)
- Number: Number for configuration purposes (settings)
- Stream: Stream of bytes (input, output)

## Multiple runs

Instead of streamed I/O, can be run in a mode where multiple instances of the code can run at once

## Staging modes

1. Single container: Takes code, I/O streams, and other stuff, and runs it in one container
2. Multi container: Takes code and other stuff, identical across all cases, and I/O strings and other stuff unique to each case, and runs the cases in different containers simultaneously, to try multiple cases

## Staging schema

Types of packets that the conductor can receive:

1. Run code
2. Stream input

Types of packets that the conductor can send:

1. Finish code
2. Stream output

"Run code" packets follow a schema specified in the language configuration. They always include language, variant, and code fields. The schema can contain fields of the following types:

- String
- Array
- Environment
- Optional string
- Stream (initial contents and/or close instruction)
- Choice

In a multi-instance run, these will be split into one group which is consistent and one which changes per case.

## JSON sources

A `string-source` can be any of:

- A raw string
- An object with a `src` property containing an ID
- An array of string sources to concat
- An object with a `type` property which is one of the following functions:
  - `join`: Takes an array (`strings`) and a string (`with`) to join with
  - `trim`: Takes a string (`string`) and an optional `trim` OR `trimStart` and/or `trimEnd` property(ies) containing strings to trim (defaults to ASCII whitespace: space, tab, newline, carriage return, and form feed)
  - `subs`: Takes a string (`string`) and an array (`substitutions`) of string substitutions to perform
  - `slice`: Takes a string (`string`), and a lower bound (`from`) and/or an upper bound (`to`) containing a signed ints to slice between
  - `condition`: Takes a condition (`condition`) and chooses between two string sources (`pass` and `fail`) based on it
- An object with a `stream` property containing an ID (will block until stream ends, unless `nonBlocking` or `stop` is set to true)

An `array-source` can be any of:

- A raw array of string sources
- An object with a `src` property containing an ID
- An object with a `type` property which is one of the following functions:
  - `concat`: Takes an array of arrays (`arrays`) and concats
  - `split`: Takes a string (`string`) and a substring (`by`) to split by
  - `filter`: Chooses strings from `array` according to a condition
  - `map`: Takes an array (`array`) and a string source (`to`) which is evaluated for each item, with an ID (`id`) being replaced by the current item (e.g., `{ type: "map", "array": ["a", "b", "c"], "id": "_", "to": [{ src: "_" }, "+"] }` would result in `["a+", "b+", "c+"]`)
  - `subs`: Takes an array (`array`) and an array (`substitutions`) of item substitutions to perform
  - `condition`: Takes a condition (`condition`) and chooses between two array sources (`pass` and `fail`) based on it
  - `zipEnvironment`: Takes an environment (`env`) and a string source, which takes two IDs (`keyId` and `valueId`) and produces strings
  - `zip`: Takes two arrays (`array1` and `array2`) and a string source, and maps to a single array with IDs (`id1` and `id2`)
  - `apply`: Applies a function to a particular set of items or slices
  - `slice`: Takes an array (`array`), and a lower bound (`from`) and/or an upper bound (`to`) containing a signed ints to slice between
  - `sort`: Takes an array (`array`) and (optionally) one of: `desc` (bool) and/or `key` (string source) OR `compare` (condition)