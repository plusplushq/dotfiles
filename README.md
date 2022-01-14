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




### Handle different file locations on different systems with the same contents (also if for example, VSCode settings are in diff folders per machine)

If you want to have the same file contents in different locations on different
systems, but maintain only a single file in your source state, you can use
a shared template.

Create the common file in the `.chezmoitemplates` directory in the source state. For
example, create `.chezmoitemplates/file.conf`. The contents of this file are
available in templates with the `template *name* .` function where *name* is the
name of the file (`.` passes the current data to the template code in `file.conf`;
see https://pkg.go.dev/text/template#hdr-Actions for details).

Then create files for each system, for example `Library/Application
Support/App/file.conf.tmpl` for macOS and `dot_config/app/file.conf.tmpl` for
Linux. Both template files should contain `{{- template "file.conf" . -}}`.

Finally, tell chezmoi to ignore files where they are not needed by adding lines
to your `.chezmoiignore` file, for example:

```
{{ if ne .chezmoi.os "darwin" }}
Library/Application Support/App/file.conf
{{ end }}
{{ if ne .chezmoi.os "linux" }}
.config/app/file.conf
{{ end }}
```

---

## Manage machine-to-machine differences

---

### Use templates

For example, your home `~/.gitconfig` on your personal machine might look like:

```toml
[user]
    email = "me@home.org"
```

Whereas at work it might be:

```toml
[user]
    email = "firstname.lastname@company.com"
```

To handle this, on each machine create a configuration file called
`~/.config/chezmoi/chezmoi.yaml` defining variables that might vary from machine
to machine. For example, for your home machine:

```yaml
 data
    personalEmail = "me@home.org"
    workEmail = "me@home.org"
```


### Use Bitwarden

chezmoi includes support for [Bitwarden](https://bitwarden.com/) using the
[Bitwarden CLI](https://github.com/bitwarden/cli) to expose data as a template
function.

Log in to Bitwarden using:

```console
$ bw login <bitwarden-email>
```

Unlock your Bitwarden vault:

```console
$ bw unlock
```

Set the `BW_SESSION` environment variable, as instructed.

The structured data from `bw get` is available as the `bitwarden` template
function in your config files, for example:

```
username = {{ (bitwarden "item" "example.com").login.username }}
password = {{ (bitwarden "item" "example.com").login.password }}
```

Custom fields can be accessed with the `bitwardenFields` template function. For
example, if you have a custom field named `token` you can retrieve its value
with:

```
{{ (bitwardenFields "item" "example.com").token.value }}
```

---
