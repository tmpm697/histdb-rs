# histdb-rs

[![Build Status](https://github.com/AlexanderThaller/histdb-rs/workflows/Rust/badge.svg?branch=main)](https://github.com/AlexanderThaller/histdb-rs/actions?query=workflow%3ARusteain)
[![crates.io](https://img.shields.io/crates/v/histdb-rs.svg)](https://crates.io/crates/histdb-rs)

Better history management for zsh. Based on ideas from
[https://github.com/larkery/zsh-histdb](https://github.com/larkery/zsh-histdb).

Licensed under MIT.

It was mainly written because the sqlite merging broke a few to many times for
me and using a sqlite database seemed overkill.

The tool is just writing CSV files for each host which makes syncing them via
git pretty painless.

Has pretty much the same feature set as zsh-histdb:

* Start and stop time of the command
* Working directory in which the command was run
* Hostname of the machine the command was run in
* Unique session ids based on UUIDs
* Exit status of the command
* Import from zsh histfile and zsh-histdb sqlite database

## Installation

You can either install the right binary from the releases page or run:

```
cargo install histdb-rs
```

## Archlinux

Install from the aur: https://aur.archlinux.org/packages/histdb-rs/

## First Start

After you installed histdb-rs you need to start the server:

```
histdb-rs server
```

By default the server will run in the foreground.

To stop the server you can run the following:

```
histdb-rs stop
```

Or send SIGTERM/SIGINT (Ctrl+C) to stop the server.

You can also use the systemd unit file in
[`histdb-rs.service`](resources/histdb-rs.service) which you can copy to
`"$HOME/.config/systemd` and enable/start with the following:

```
systemctl --user daemon-reload
systemctl --user enable histdb-rs.service
systemctl --user start histdb-rs.service
```

After that you can add the following to your `.zshrc` to enable histdb-rs for
you shell.

```
eval "$(histdb-rs init)"
```

You can run that in your current shell to enable histdb-rs or restart your
shell.

## Usage

Help output of default command:

```
histdb-rs 2.0.0

USAGE:
    histdb-rs [FLAGS] [OPTIONS] [SUBCOMMAND]

FLAGS:
        --all-hosts
            Print all hosts

        --disable-formatting
            Disable fancy formatting

        --filter-failed
            Filter out failed commands (return code not 0)

    -h, --help
            Prints help information

        --hide-header
            Disable printing of header

    -i, --in
            Only print entries that have been executed in the current directory

        --no-subdirs
            Exclude subdirectories when filtering by folder

        --show-duration
            Show how long the command ran

        --show-host
            Print host column

        --show-pwd
            Show directory in which the command was run

        --show-session
            Show session id for command

        --show-status
            Print returncode of command

    -V, --version
            Prints version information


OPTIONS:
    -c, --command <command>
            Only print entries beginning with the given command

    -t, --text <command-text>
            Only print entries containing the given regex

        --config-path <config-path>
            Path to the socket for communication with the server [env: HISTDBRS_CONFIG_PATH=]  [default:
            /home/athaller/.config/histdb-rs/config.toml]
    -d, --data-dir <data-dir>
            Path to folder in which to store the history files [default: /home/athaller/.local/share/histdb-rs]

    -e, --entries-count <entries-count>
            How many entries to print [default: 25]

        --find-status <find-status>
            Find commands with the given return code

    -f, --folder <folder>
            Only print entries that have been executed in the given directory

        --hostname <hostname>
            Filter by given hostname

        --session <session>
            Filter by given session


SUBCOMMANDS:
    bench
            Run benchmark against server

    disable
            Disable history recording for current session

    enable
            Enable history recording for current session

    help
            Prints this message or the help of the given subcommand(s)

    import
            Import entries from existing histdb sqlite or zsh histfile

    init
            Print out shell functions needed by histdb and set current session id

    precmd
            Finish command for current session

    server
            Start the server

    session_id
            Get new session id

    stop
            Stop the server

    zshaddhistory
            Add new command for current session
```

The most basic command ist just running `histdb-rs` without any arguments:

```
» histdb-rs
 tmn    cmd
 14:28  cargo +nightly install --path .
```

That will print the history for the current machine. By default only the last
25 entries will be printed.

## Git

Histdb-rs was written to easily sync the history between multiple machines. For
that histdb-rs will write separate history files for each machine.

If you want to sync between machines go to the datadir (default is
`$HOME/.local/share/histdb-rs`) and run the following commands:

```
git init
git add :/
git commit -m "Initial commit"
```

After that you can configure origins and start syncing the files between
machines. There is no autocommit/autosync implemented as we don't want to have
commits for each command run. This could be changed in the future.

## Configuration

There is also a way to configure `histdb-rs`. By default the configuration is stored under `$HOME/.config/histdb-rs/config.toml`. A different path can be specified using the `--config-path` option.

The default configuration looks like this:

```toml
# When true will not save commands that start with a space.
# Default: true
ignore_space = true

# The log level to run under.
# Default: Warn
log_level = "Warn"
```

## Import

### zsh-histdb

```
» histdb import histdb -h
histdb-rs-import-histdb 0.1.0
Import entries from existing histdb sqlite file

USAGE:
    histdb-rs import histdb [OPTIONS]

FLAGS:
    -h, --help
            Prints help information


OPTIONS:
    -d, --data-dir <data-dir>
            Path to folder in which to store the history files [default: $HOME/.local/share/histdb-rs]

    -i, --import-file <import-file>
            Path to the existing histdb sqlite file [default: $HOME/.histdb/zsh-history.db]
```

If the defaults for the `data-dir` and the `import-file` are fine you can just
run the following command:

```
histdb import histdb
```

This will create CSV files for each `hostname` found in the sqlite database. It
will create a UUID for each unique session found in sqlite so command run in the
same session should still be grouped together.

### zsh histfile

```
» histdb import histfile -h
histdb-rs-import-histfile 0.1.0
Import entries from existing zsh histfile

USAGE:
    histdb-rs import histfile [OPTIONS]

FLAGS:
    -h, --help
            Prints help information


OPTIONS:
    -d, --data-dir <data-dir>
            Path to folder in which to store the history files [default: $HOME/.local/share/histdb-rs]

    -i, --import-file <import-file>
            Path to the existing zsh histfile file [default: $HOME/.histfile]
```

If the defaults for the `data-dir` and the `import-file` are fine you can just
run the following command:

```
histdb import histfile
```

As the information stored in the histfile is pretty limited the following
information will be stored:

* `time_finished` will be parsed from the histfile
* `result` (exit code) will be parsed from the histfile
* `command` will be parsed from the histfile
* `time_start` will be copied over from `time_finished`
* `hostname` will use the current machines hostname
* `pwd` will be set to the current users home directory
* `session_id` will be generated and used for all commands imported from the
histfile
* `user` will use the current user thats running the import

## Contribution

I'm happy with how the tool works for me so I won't expand it further but
contributions for features and fixes are always welcome!

## Alternatives

* https://github.com/ellie/atuin
