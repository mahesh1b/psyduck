---
title: "Dotfiles"
layout: "dotfiles"
url: /dot-files/
summary: "dot-files"
---

### .Zshrc
```sh
# Path to your Oh My Zsh installation.
export ZSH="$HOME/.oh-my-zsh"

# See https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
ZSH_THEME="spaceship"

# Add wisely, as too many plugins slow down shell startup.
plugins=(git zsh-autosuggestions zsh-syntax-highlighting aws terraform)

# kubectl autocompletion
autoload -Uz compinit
compinit
source <(kubectl completion zsh)

# DuploToken
example() {
  export duplo_token=$(duplo-jit duplo --host https://example.com --interactive | jq -r '.DuploToken')
}

# Netbird Connnect
netbird_connect(){
  netbird down
  netbird up --management-url https://example.com:33073 --admin-url https://example.com
}

# Put individual kubeconfig files in ~/.kube/custom-contexts dir
# Function to append all kubeconfig files in the "config" directory to KUBECONFIG
init_kubeconfig() {
    local kubeconfig_dir="$HOME/.kube/custom-contexts"
    local kubeconfigs="$HOME/.kube/config"

    for file in "$kubeconfig_dir"/*; do
        if [ -f "$file" ]; then
            kubeconfigs="$kubeconfigs:$file"
        fi
    done

    # Remove the leading colon if any files were found
    if [ -n "$kubeconfigs" ]; then
        kubeconfigs="${kubeconfigs#:}"
        export KUBECONFIG="${KUBECONFIG:+$KUBECONFIG:}$kubeconfigs"
    fi
}

source $ZSH/oh-my-zsh.sh

export PATH=/Users/mahesh/.local/bin:$PATH

init_kubeconfig

# Aliases
alias tf=terraform
alias k=kubectl
alias speedtest='wget -O /dev/null http://speed.transip.nl/100mb.bin'
alias ls='eza -lah --no-time --icons --git --group-directories-first --no-permissions'
alias ld='eza -lahD --no-time --icons --git --no-permissions --no-filesize'
alias vim=nvim
```

### Ghostty Config
```conf
# Theme
term = xterm-256color
theme = ayu
title = " "

# Fonts
font-family = "JetBrainsMono Nerd Font Propo"
font-size = 15
font-thicken = true

# Mouse, Cursor & Clipboard
cursor-style = block
copy-on-select = true
cursor-style-blink = true
mouse-hide-while-typing = true

# Window appearance
bold-is-bright = true
window-padding-y = 3,4
window-padding-x = 6
resize-overlay = never
background-opacity = 1
window-decoration = true
background-blur-radius = 20
window-padding-balance = true
macos-window-shadow = true
macos-titlebar-style = tabs
window-height = 30
window-width = 90
macos-titlebar-proxy-icon = hidden
#macos-option-as-alt = true

# general
confirm-close-surface = false

# Splits
keybind = ctrl+shift+v=new_split:right
keybind = ctrl+shift+h=new_split:down
keybind = ctrl+shift+,=previous_tab
keybind = ctrl+shift+.=next_tab

# Clipboard & Font
keybind = super+c=copy_to_clipboard
keybind = super+v=paste_from_clipboard
keybind = ctrl+shift+plus=increase_font_size:1
keybind = ctrl+shift+minus=decrease_font_size:1

# Windows
keybind = super+q=quit
keybind = super+t=new_tab
keybind = super+n=new_window
keybind = super+k=clear_screen
keybind = ctrl+shift+q=close_surface

# Services
keybind = ctrl+shift+r=reload_config
```

### k9s Ayu Theme
```yaml
# -----------------------------------------------------------------------------
# K9s Ayu Terminal Matched Theme
# Based on the exact terminal colors provided
# -----------------------------------------------------------------------------

# Terminal colors
foreground: &foreground "#e6e1cf"
background: &background "#0f1419"
current_line: &current_line "#1c242b"
selection: &selection "#253340"
comment: &comment "#626A73"
black: &black "#000000"
red: &red "#ff3333"
bright_red: &bright_red "#ff6565"
green: &green "#b8cc52"
bright_green: &bright_green "#eafe84"
yellow: &yellow "#e7c547"
bright_yellow: &bright_yellow "#fff779"
blue: &blue "#36a3d9"
bright_blue: &bright_blue "#68d5ff"
magenta: &magenta "#f07178"
bright_magenta: &bright_magenta "#ffa3aa"
cyan: &cyan "#95e6cb"
bright_cyan: &bright_cyan "#c7fffd"
white: &white "#ffffff"
cursor: &cursor "#f29718"

k9s:
  body:
    fgColor: *foreground
    bgColor: *background
    logoColor: *blue
  prompt:
    fgColor: *foreground
    bgColor: *background
    suggestColor: *cursor
  info:
    fgColor: *magenta
    sectionColor: *foreground
  help:
    fgColor: *foreground
    bgColor: *background
    keyColor: *magenta
    numKeyColor: *blue
    sectionColor: *green
  dialog:
    fgColor: *foreground
    bgColor: *background
    buttonFgColor: *foreground
    buttonBgColor: *magenta
    buttonFocusFgColor: *background
    buttonFocusBgColor: *cyan
    labelFgColor: *cursor
    fieldFgColor: *foreground
  frame:
    border:
      fgColor: *selection
      focusColor: *current_line
    menu:
      fgColor: *foreground
      keyColor: *magenta
      numKeyColor: *magenta
    crumbs:
      fgColor: *foreground
      bgColor: *comment
      activeColor: *blue
    status:
      newColor: *cyan
      modifyColor: *blue
      addColor: *green
      errorColor: *red
      highlightColor: *cursor
      killColor: *comment
      completedColor: *comment
    title:
      fgColor: *foreground
      bgColor: *background
      highlightColor: *cursor
      counterColor: *blue
      filterColor: *magenta
  views:
    charts:
      bgColor: *background
      defaultDialColors:
        - *blue
        - *red
      defaultChartColors:
        - *blue
        - *red
    table:
      fgColor: *foreground
      bgColor: *background
      cursorFgColor: *background
      cursorBgColor: *blue
      header:
        fgColor: *foreground
        bgColor: *background
        sorterColor: *selection
    xray:
      fgColor: *foreground
      bgColor: *background
      cursorColor: *current_line
      graphicColor: *blue
      showIcons: false
    yaml:
      keyColor: *magenta
      colonColor: *blue
      valueColor: *foreground
    logs:
      fgColor: *foreground
      bgColor: *background
      indicator:
        fgColor: *foreground
        bgColor: *background
```