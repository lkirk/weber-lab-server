# Conventions

## Command notation

Throughout the documentation, I will list out commands to be passed in to the
command line (bash, sh, zsh, etc...). All commands are shell agnostic unless
specified

In some cases, I will provide command line arguments that need to be filled in
by the user. I will denote them as `<option>`. Do not keep the `<` or `>` in the
command as they will mess up the command.

For example if you want list all files in a directory:
```
ls -la <some_directory>
```

Say that you want to list files in the `/tmp` dir, you will write:
```
ls -la /tmp
```

## User Privileges

All commands that are meant to be run as the user will be prefixed with `$`. All
commands meant to be run as root will be prefixed with `#`.

This command should be run as the user:
```
$ ls -la <some_directory>
```

And this command should be run as root:

```
# ls -la <some_directory>
```

## Key Commands

Commands prefixed with `C-` mean control. For instance `C-x` means press
control-x.
