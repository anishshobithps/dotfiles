<img src="n10nce.svg" alt="n10nce" align="left" width="72" />

# dotfiles

macOS dotfiles managed with [chezmoi](https://www.chezmoi.io/). One command sets up a fresh Mac with a signed-commit git config, a modern zsh shell, pinned language runtimes, and a tiling window manager.

<br clear="left" />

## What's included

| Tool | Role |
| --- | --- |
| chezmoi | Manages and applies these dotfiles |
| starship | Shell prompt (Tokyo Night theme) |
| mise | Runtime version manager (Node, Python) |
| fzf, zoxide, fd, ripgrep | Fuzzy finding, smart `cd`, fast search |
| eza, bat | Modern `ls` and `cat` replacements |
| git-delta | Syntax-highlighted git diffs |
| zsh-autosuggestions, zsh-syntax-highlighting, zsh-completions | Shell autocomplete stack |
| AeroSpace | i3-style tiling window manager |
| borders | Focus borders for tiled windows |
| Zed, Raycast | Editor and launcher |

## Layout

```
.
├── .chezmoiroot            # points chezmoi at home/
└── home/
    ├── .chezmoi.toml.tmpl  # prompts for name and email on init
    ├── .chezmoiignore      # keeps Brewfile in source, out of $HOME
    ├── .chezmoiscripts/
    │   └── run_onchange_install-packages.sh.tmpl
    ├── Brewfile            # package list for brew bundle
    ├── dot_gitconfig.tmpl  # git config with SSH commit signing
    ├── dot_zprofile.tmpl   # Homebrew shell env (macOS)
    ├── dot_zshrc.tmpl      # shell config, aliases, fzf integration
    └── dot_config/
        ├── aerospace/aerospace.toml
        ├── mise/config.toml
        └── starship.toml
```

`dot_` maps to a leading dot, so `dot_gitconfig.tmpl` becomes `~/.gitconfig`. The `.tmpl` files are templates that fill in per-machine values.

## Fresh machine setup

Run these in order on a new Mac. The order matters, because the chezmoi install script calls Homebrew.

### 1. Xcode command line tools and Homebrew

```sh
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Add Homebrew to the shell after it installs:

```sh
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 2. SSH key for auth and commit signing

```sh
ssh-keygen -t ed25519 -C "your-github-email"
```

Create `~/.ssh/config`:

```sh
mkdir -p ~/.ssh && cat > ~/.ssh/config <<'EOF'
Host github.com
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519
EOF
```

Load the key and copy the public half:

```sh
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
pbcopy < ~/.ssh/id_ed25519.pub
```

Add that key to GitHub twice, under Settings → SSH and GPG keys → New SSH key: once as an **Authentication Key**, once as a **Signing Key**. Then test:

```sh
ssh -T git@github.com
```

### 3. Signer verification file

The git config verifies signatures against this file. It holds your public key, so it stays local and is not in the repo:

```sh
mkdir -p ~/.config/git
echo "your-github-email namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" > ~/.config/git/allowed_signers
```

Use the same email you will enter in the next step.

### 4. Install chezmoi and apply

```sh
sh -c "$(curl -fsLS https://get.chezmoi.io)" -- init --apply --ssh anishshobithps
```

This clones the repo, prompts for your name and email, applies every config, and runs `brew bundle` to install the full package list. Node 24 and Python 3.14 install through mise.

### 5. Terminal font

The Tokyo Night prompt and file icons need a Nerd Font:

```sh
brew install --cask font-meslo-lg-nerd-font
```

Set the terminal font to **MesloLGS Nerd Font** (Terminal → Settings → Profiles → Text → Change).

### 6. AeroSpace permission

Launch AeroSpace and grant Accessibility access under System Settings → Privacy & Security → Accessibility. It starts tiling once enabled, and runs at login from then on.

## Daily workflow

Edit a config through chezmoi so the source stays in sync:

```sh
chezmoi edit ~/.zshrc
chezmoi diff
chezmoi apply
```

Add a new file to management:

```sh
chezmoi add ~/.config/some/file
```

Push changes:

```sh
chezmoi cd
git add . && git commit -m "message"
git push
exit
```

Commit signing is on, so verify a commit with:

```sh
git log --show-signature -1
```

## Language runtimes

mise pins global defaults in `home/dot_config/mise/config.toml`:

```toml
[tools]
node = "24"
python = "3.14"
```

Node 24 is the current LTS. Python has no LTS, so 3.14 is the latest stable line. The pins track the newest patch within each line.

For a per-project version, run this inside the project. mise writes a local `mise.toml` and switches automatically on `cd`:

```sh
mise use node@22
```

## AeroSpace keybindings

The modifier `alt` is the Option key.

| Keys | Action |
| --- | --- |
| Option + H / J / K / L | Move focus left / down / up / right |
| Option + Shift + H / J / K / L | Move the window |
| Option + − / = | Shrink / grow the window |
| Option + / | Tiling layout |
| Option + , | Accordion layout |
| Option + 1–5 | Switch to workspace |
| Option + Shift + 1–5 | Send window to workspace |
| Option + Tab | Previous workspace |
| Option + Shift + ; | Service mode |

In service mode: `Esc` reloads the config, `R` re-tiles a workspace, `F` toggles floating, `Backspace` closes all windows but the focused one.

AeroSpace uses its own workspaces, not native macOS Spaces, so the three-finger swipe does not switch AeroSpace workspaces. Use Option + 1–5.

## Notes

The `borders` install needs a third-party tap that Homebrew blocks by default. The install script runs `brew tap` and `brew trust felixkratz/formulae` before `brew bundle`, so a fresh setup installs it without manual steps.

The Brewfile lives in `.chezmoiignore`. It stays in the source tree for the install script to read, and never gets copied into `$HOME`.

Re-running the install script is automatic. It carries a hash of the Brewfile, so editing the package list retriggers `brew bundle` on the next `chezmoi apply`.
