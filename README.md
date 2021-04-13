# Taskfile

This repository contains the default Taskfile template for getting started in your own projects. A Taskfile is a bash (or zsh etc.) script that follows a specific format. It's called `Taskfile`, sits in the root of your project and contains the tasks to build your project.

This is a fork of the original Taskfile project. It's been adapted for a Python workflow and has some extra utilities to make things easier.

```sh
#!/usr/bin/env bash

set -o errexit
shopt -s inherit_errexit 2>/dev/null || true
set -o pipefail
set -o nounset

# full path current folder
__dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# full path of the script.sh (including the name)
__file="${__dir}/$(basename "${BASH_SOURCE[0]}")"
# name of the script
__base="$(basename ${__file} .sh)"
# full path of the parent folder
__root="$(cd "$(dirname "${__dir}")" && pwd)" # <-- change this as it depends on your app

export appdir="$__dir"/
export SOURCE_FILES="$appdir tests"

function assert_env {
    source "$__dir"/".venv/bin/activate" || exit 1
    echo "Pip location:"
    pip_cmd=$(command -v pip)
    echo "$pip_cmd"

    current=$(pwd)
    pip_path="$current/.venv/bin/pip"
    echo "$pip_path"

    if [[ "$pip_cmd" -ef "$pip_path" ]]; then
        echo "paths match"
    else
        exit 1
    fi
}

function clean {
    log "cleaning files"
    find . -name '__pycache__' -exec rm -fr {} +;
    find . -name '.ipynb_checkpoints' -exec rm -fr {} +;
    find . -name '*.pyo' -exec rm -f {} +;
    find . -name '*.pyc' -exec rm -f {} +;
    find . -name '*.egg-info' -exec rm -fr {} +;
    find . -name '*~' -exec rm -f {} +;
    find . -name '*.egg' -exec rm -f {} +
}

function build {
    echo "build task not implemented"
}

function deps {
    assert_env
    pip install pip-tools pip setuptools
    pip-compile -v --build-isolation --generate-hashes --allow-unsafe --output-file requirements/main.txt requirements/main.in && \
    pip-compile -v --build-isolation --generate-hashes --allow-unsafe --output-file requirements/dev.txt requirements/dev.in
}

function server {
    echo "running app with uvicorn"
    assert_env
    uvicorn --workers 2 app.main:app --reload --reload-dir "$appdir"
}


function default {
    # Default task to execute
    clean
}

function tasks {
    echo "$0 <task> <args>"
    echo "Tasks:"
    compgen -A function | cat -n
}

usage() { grep '^#/' "$0" | cut --characters=4-; exit 0; }
REGEX='(^|\W)(-h|--help)($|\W)'
[[ "$*" =~ $REGEX ]] && usage || true
TIMEFORMAT="Task completed in %3lR"
time ${@:-default}

```

And to run a task:

    $ ./Taskfile clean

## Install

To "install", add the following to your `.bashrc` or `.zshrc` (or `.whateverrc`):

    # Quick start with the default Taskfile template
    alias run-init="curl -so Taskfile https://raw.githubusercontent.com/polyrand/Taskfile/master/Taskfile && chmod +x Taskfile"
    
    # Run your tasks like: run <task>
    alias run=./Taskfile

## Usage

Open your directory and run `run-init` to add the default Taskfile template to your project directory:

    $ cd my-project
    $ run-init

Open the `Taskfile` and add your tasks. To run tasks, use `run`:

```bash
+$ run tasks
[Taskfile] ==================== BEGIN ====================
./Taskfile <task> <args>
Tasks:
     1	assert_env
     2	build
     3	buildprod
     4	clean
     5	default
     6	deps
	  .
	  .
Task completed in 0m0.008s
[Taskfile] SUCCESS
[Taskfile] #################### END ####################
```

## Techniques

### Arguments

Let’s pass some arguments to a task. Arguments are accessible to the task via the `$1, $2, $n..` variables. Let’s allow us to specify the port of the HTTP server:

```sh
#!/usr/bin/env bash

function serve {
  python -m SimpleHTTPServer $1
}

"$@"
```

And if we run the `serve` task with a new port:

    $ ./Taskfile serve 9090
    Serving HTTP on 0.0.0.0 port 9090 ...

### Extending `PATH`

You can temporarily modify your `PATH` during tasks execution by modifying it at the beginning of your script.

```sh
#!/usr/bin/env bash
PATH=./node_modules/.bin:$PATH

# ...
# ...
```

### Task Dependencies

Sometimes tasks depend on other tasks to be completed before they can start. To add another task as a dependency, simply call the task's function at the top of the dependant task's function.

```sh
#!/usr/bin/env bash

function clean {
  rm -r build dist
}

function build {
  webpack src/index.js --output-path build/
}

function minify {
  uglify build/*.js dist/
}

function deploy {
  clean && build && minify
  scp dist/index.js sergey@google.com:/top-secret/index.js
}

"$@"
```

### Parallelisation

To run tasks in parallel, you can us Bash’s `&` operator in conjunction with `wait`. The following will build the two tasks at the same time and wait until they’re completed before exiting.

```bash
#!/bin/bash

function build {
    echo "beep $1 boop"
    sleep 1
    echo "built $1"
}

function build-all {
    build web & build mobile &
    wait
}

"$@"
```

And execute the `build-all` task:

    $ run build-all
    beep web boop
    beep mobile boop
    built web
    built mobile

### Default task

To make a task the default task called when no arguments are passed, we can use bash’s default variable substitution `${VARNAME:-<default value>}` to return `default` if `$@` is empty. 

```sh
#!/bin/bash

function build {
    echo "beep boop built"
}

function default {
    build
}

"${@:-default}"
```

Now when we run `./Taskfile`, the `default` function is called.


### Runtime Statistics

To add some nice runtime statistics so you can keep an eye on build times, we use the built in `time` and pass if a formatter.

```bash
#!/bin/bash

function build {
    echo "beep boop built"
    sleep 1
}

function default {
    build
}

TIMEFORMAT="Task completed in %3lR"
time ${@:-default}
```

And if we execute the `build` task:

    $ ./Taskfile build 
    beep boop built 
    Task completed in 0m1.008s

### Help

The final addition I recommend adding to your base Taskfile is the  task which emulates, in a much more basic fashion,  (with no arguments). It prints out usage and the available tasks in the Taskfile to show us what tasks we have available to ourself.

The `compgen -A function` is a [bash builtin](https://www.gnu.org/software/bash/manual/html_node/Programmable-Completion-Builtins.html) that will list the functions in our Taskfile (i.e. tasks). This is what it looks like when we run the  task:

    $ ./Taskfile tasks
    ./Taskfile <task> <args>
    Tasks:
         1  build
         2  default
         3  help
    Task completed in 0m0.005s
    
The Taskfile also implements another help syntax. You can use the `-h`\`--help` flags. When you do it, it will read all the comments starting with `#/` and print them:

```bash
#!/usr/bin/env bash
#/ PATH=./node_modules/.bin:$PATH
#/ https://www.tldp.org/LDP/abs/html/options.html

# ...
# ...
# ...

usage() { grep '^#/' "$0" | cut --characters=4-; exit 0; }
REGEX='(^|\W)(-h|--help)($|\W)'
[[ "$*" =~ $REGEX ]] && usage || true
TIMEFORMAT="Task completed in %3lR"
time ${@:-default}
```

Running it:

```bash
$ run --help
[Taskfile] ==================== BEGIN ====================
PATH=./node_modules/.bin:$PATH
https://www.tldp.org/LDP/abs/html/options.html
[Taskfile] SUCCESS
[Taskfile] #################### END ####################
```

### Extra features

The Taskfile includes a lot of standard bash utils to play with:

* Variable for: current folder, Taskfil path, application folder, etc.

Some interesting variables to set/modify:

`export appdir="$__dir"/app`: This is used to specify where the main application folder is. The example here is a folder called `app/`

`export SOURCES_FILES="$appdir tests"`: this can be used to include multiple folders. For example here I'm using the `appdir` variable we saw before and another folder called `tests/`. You can use this for example for linting files in multiple folders but doing other tasks only in the `$appdir` folder.

* Logging. Use `log "Message"` anywhere in your taskfile. Our without a message to enable command tracing.
* Exit trap
* Good defaults


*Some of those best-practices come from [here](https://kvz.io/bash-best-practices.html)*

### `task:` namespace

If you find you need to breakout some code into reusable functions that aren't tasks by themselves and don't want them cluttering your `help` output, you can introduce a namespace to your task functions. Bash is pretty lenient with it's function names so you could, for example, prefix a task function with  `task:`. Just remember to use that namespace when you're calling other tasks and in your `task:$@` entrypoint!

```sh
#!/bin/bash
PATH=./node_modules/.bin

function task:build-web {
    build-target web
}

function task:build-desktop {
    build-target desktop
}

function build-target {
    BUILD_TARGET=$1 webpack --production
}

function task:default {
    task:help
}

function task:help {
    echo "$0 <task> <args>"
    echo "Tasks:"

    # We pick out the `task:*` functions
    compgen -A function | sed -En 's/task:(.*)/\1/p' | cat -n
}

TIMEFORMAT="Task completed in %3lR"
time "task:${@:-default}"
```

### Executing tasks

So typing out `./Taskfile` every time you want to run a task is a little lousy.  just flows through the keyboard so naturally that I wanted something better. The solution for less keystrokes was dead simple: add an alias for `run` (or `task`, whatever you fancy) and stick it in your *.zshrc.* Now, it now looks the part.

    $ alias run=./Taskfile
    $ run build
    beep boop built
    Task completed in 0m1.008s

### Quickstart

Alongside my `run` alias, I also added a `run-init` to my *.zshrc* to quickly get started with a new Taskfile in a project. It downloads a [small Taskfile template](https://github.com/polyrand/Taskfile) to the current directory and makes it executable:

    $ alias run-init="curl -so Taskfile https://raw.githubusercontent.com/polyrand/Taskfile/master/Taskfile && chmod +x Taskfile && chmod +x Taskfile"

    $ run-init
    $ run build
    beep boop built
    Task completed in 0m1.008s

### Free Features

* Conditions and loops. Bash and friends have support for conditions and loops so you can error if parameters aren’t passed or if your build fails.
* Streaming and piping. Don’t forget, we’re in a shell and you can use all your favourite redirections and piping techniques.
* All your standard tools like `rm` and `mkdir`.
* Globbing. Shells like zsh can expand globs like `**/*.js` for you automatically to pass to your tools.
* Environment variables like `NODE_ENV` are easily accessible in your Taskfiles.

#### Considerations

When writing my Taskfile, these are some considerations I found useful:

* You should try to use tools that you know users will have installed and working on their system. I’m not saying you have to be POSIX.1 compliant but be weary of using tools that aren’t standard (or difficult to install).
* Keep it pretty. The reason for the Taskfile format is to keep your tasks organised and readable.

#### Caveats

The only caveat with the Taskfile format is we forgo compatibility with Windows which sucks. Of course, users can install Cygwin but one of most attractive things about the Taskfile format is not having to install external software to run the tasks. Hopefully, [Microsoft’s native bash shell in Windows 10](http://www.howtogeek.com/249966 how-to-install-and-use-the-linux-bash-shell-on-windows-10/) can do work well for us in the future.

*****

### Collaboration

The Taskfile format is something I’d love to see become more widespread and it’d be awesome if we could all come together on a standard of sorts. Things like simple syntax highlighting extensions or best practices guide would be awesome to formalise.
