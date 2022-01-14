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


### Install packages with scripts

Change to the source directory and create a file called
`run_once_install-packages.sh`:

```console
$ chezmoi cd
$ $EDITOR run_once_install-packages.sh
```

In this file create your package installation script, e.g.

```sh
#!/bin/sh
sudo apt install ripgrep
```

The next time you run `chezmoi apply` or `chezmoi update` this script will be
run. As it has the `run_once_` prefix, it will not be run again unless its
contents change, for example if you add more packages to be installed.

This script can also be a template. For example, if you create
`run_once_install-packages.sh.tmpl` with the contents:

```
{{ if eq .chezmoi.os "linux" -}}
#!/bin/sh
sudo apt install ripgrep
{{ else if eq .chezmoi.os "darwin" -}}
#!/bin/sh
brew install ripgrep
{{ end -}}
```

This will install `ripgrep` on both Debian/Ubuntu Linux systems and macOS.

---


---

### Run a script when the contents of another file changes (could be useful for your list of packages to install, whenever that changes for example)

chezmoi's `run_` scripts are run every time you run `chezmoi apply`, whereas
`run_once_` scripts are run only when their contents have changed, after
executing them as templates. You use this to cause a `run_once_` script to run
when the contents of another file has changed by including a checksum of the
other file's contents in the script.

For example, if your [dconf](https://wiki.gnome.org/Projects/dconf) settings are
stored in `dconf.ini` in your source directory then you can make `chezmoi apply`
only load them when the contents of `dconf.ini` has changed by adding the
following script as `run_once_dconf-load.sh.tmpl`:

## Use chezmoi on macOS (and maybe linux since Linuxbrew is available)

### Use `brew bundle` to manage your brews and casks

Homebrew's [`brew bundle`
subcommand](https://docs.brew.sh/Manpage#bundle-subcommand) allows you to
specify a list of brews and casks to be installed. You can integrate this with
chezmoi by creating a `run_once_` script. For example, create a file in your
source directory called `run_once_before_install-packages-darwin.sh.tmpl`
containing:

```
{{- if (eq .chezmoi.os "darwin") -}}
#!/bin/bash

brew bundle --no-lock --file=/dev/stdin <<EOF
brew "git"
cask "google-chrome"
EOF
{{ end -}}
```

Note that the `Brewfile` is embedded directly in the script with a bash here
document. chezmoi will run this script whenever its contents change, i.e. when
you add or remove brews or casks.

---


## Use chezmoi on Windows

---

### Detect Windows Subsystem for Linux (WSL)

WSL can be detected by looking for the string `Microsoft` or `microsoft` in
`/proc/sys/kernel/osrelease`, which is available in the template variable
`.chezmoi.kernel.osrelease`, for example:

```
{{ if (eq .chezmoi.os "linux") }}
{{   if (.chezmoi.kernel.osrelease | lower | contains "microsoft") }}
# WSL-specific code
{{   end }}
{{ end }}
```

---



### Run a PowerShell script as admin on Windows

Put the following at the top of your script:

```powershell
# Self-elevate the script if required
if (-Not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] 'Administrator')) {
  if ([int](Get-CimInstance -Class Win32_OperatingSystem | Select-Object -ExpandProperty BuildNumber) -ge 6000) {
    $CommandLine = "-NoExit -File `"" + $MyInvocation.MyCommand.Path + "`" " + $MyInvocation.UnboundArguments
    Start-Process -FilePath PowerShell.exe -Verb Runas -ArgumentList $CommandLine
    Exit
  }
}
```

---


---

## Use chezmoi with GitHub Codespaces, Visual Studio Codespaces, or Visual Studio Code Remote - Containers

The following assumes you are using chezmoi 1.8.4 or later. It does not work
with earlier versions of chezmoi.

You can use chezmoi to manage your dotfiles in [GitHub
Codespaces](https://docs.github.com/en/github/developing-online-with-codespaces/personalizing-codespaces-for-your-account),
[Visual Studio
Codespaces](https://docs.microsoft.com/en/visualstudio/codespaces/reference/personalizing),
and [Visual Studio Code Remote -
Containers](https://code.visualstudio.com/docs/remote/containers#_personalizing-with-dotfile-repositories).

For a quick start, you can clone the [`chezmoi/dotfiles`
repository](https://github.com/chezmoi/dotfiles) which supports Codespaces out
of the box.

The workflow is different to using chezmoi on a new machine, notably:
* These systems will automatically clone your `dotfiles` repo to `~/dotfiles`,
  so there is no need to clone your repo yourself.
* The installation script must be non-interactive.
* When running in a Codespace, the environment variable `CODESPACES` will be set
  to `true`. You can read its value with the [`env` template
  function](http://masterminds.github.io/sprig/os.html).

First, if you are using a chezmoi configuration file template, ensure that it is
non-interactive when running in Codespaces, for example, `.chezmoi.toml.tmpl`
might contain:

```
{{- $codespaces:= env "CODESPACES" | not | not -}}
sourceDir = {{ .chezmoi.sourceDir | quote }}

[data]
    name = "Your name"
    codespaces = {{ $codespaces }}
{{- if $codespaces }}{{/* Codespaces dotfiles setup is non-interactive, so set an email address */}}
    email = "your@email.com"
{{- else }}{{/* Interactive setup, so prompt for an email address */}}
    email = {{ promptString "email" | quote }}
{{- end }}
```

This sets the `codespaces` template variable, so you don't have to repeat `(env
"CODESPACES")` in your templates. It also sets the `sourceDir` configuration to
the `--source` argument passed in `chezmoi init`.

Second, create an `install.sh` script that installs chezmoi and your dotfiles:

```sh
#!/bin/sh

set -e # -e: exit on error

if [ ! "$(command -v chezmoi)" ]; then
  bin_dir="$HOME/.local/bin"
  chezmoi="$bin_dir/chezmoi"
  if [ "$(command -v curl)" ]; then
    sh -c "$(curl -fsLS https://chezmoi.io/get)" -- -b "$bin_dir"
  elif [ "$(command -v wget)" ]; then
    sh -c "$(wget -qO- https://chezmoi.io/get)" -- -b "$bin_dir"
  else
    echo "To install chezmoi, you must have curl or wget installed." >&2
    exit 1
  fi
else
  chezmoi=chezmoi
fi

# POSIX way to get script's dir: https://stackoverflow.com/a/29834779/12156188
script_dir="$(cd -P -- "$(dirname -- "$(command -v -- "$0")")" && pwd -P)"
# exec: replace current process with chezmoi init
exec "$chezmoi" init --apply "--source=$script_dir"
```

Ensure that this file is executable (`chmod a+x install.sh`), and add
`install.sh` to your `.chezmoiignore` file.

It installs the latest version of chezmoi in `~/.local/bin` if needed, and then
`chezmoi init ...` invokes chezmoi to create its configuration file and
initialize your dotfiles. `--apply` tells chezmoi to apply the changes
immediately, and `--source=...` tells chezmoi where to find the cloned
`dotfiles` repo, which in this case is the same folder in which the script is
running from.

If you do not use a chezmoi configuration file template you can use `chezmoi
apply --source=$HOME/dotfiles` instead of `chezmoi init ...` in `install.sh`.

Finally, modify any of your templates to use the `codespaces` variable if
needed. For example, to install `vim-gtk` on Linux but not in Codespaces, your
`run_once_install-packages.sh.tmpl` might contain:

```
{{- if (and (eq .chezmoi.os "linux") (not .codespaces)) -}}
#!/bin/sh
sudo apt install -y vim-gtk
{{- end -}}
```

---
