# Dotfiles setup using [Chezmoi](https://www.chezmoi.io/)

## Install chezmoi and your dotfiles on a new machine with a single command
```
sh -c "$(curl -fsLS git.io/chezmoi)" -- init --apply plusplushq
```

## Chezmoi How To (Todos)
### Install chezmoi and your dotfiles on a new machine with a single command

```console
$ sh -c "$(curl -fsLS chezmoi.io/get)" -- init --apply plusplushq 
```

### Use your preferred editor with `chezmoi edit` and `chezmoi edit-config`

```console
$ export EDITOR="code --wait"
```
### Handle configuration files which are externally modified (what I was wondering about with VSCode settings)

Some programs modify their configuration files. When you next run `chezmoi
apply`, any modifications made by the program will be lost.

You can track changes to these files by replacing with a symlink back to a file
in your source directory, which is under version control. Here is a worked
example for VSCode's `settings.json` on Linux:

Copy the configuration file to your source directory:

```console
$ cp ~/.config/Code/User/settings.json $(chezmoi source-path)
```

Tell chezmoi to ignore this file:

```console
$ echo settings.json >> $(chezmoi source-path)/.chezmoiignore
```

Tell chezmoi that `~/.config/Code/User/settings.json` should be a symlink to the
file in your source directory:

```console
$ mkdir -p $(chezmoi source-path)/private_dot_config/private_Code/User
$ echo -n "{{ .chezmoi.sourceDir }}/settings.json" > $(chezmoi source-path)/private_dot_config/private_Code/User/symlink_settings.json.tmpl
```

The prefix `private_` is used because the `~/.config` and `~/.config/Code`
directories are private by default.

Apply the changes:

```console
$ chezmoi apply -v
```

Now, when the program modifies its configuration file it will modify the file in
the source state instead.


