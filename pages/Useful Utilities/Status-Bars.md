---
weight: 2
title: Status bars
---

## Waybar

Waybar is a GTK status bar made specifically for wlroots compositors and
supports Hyprland by default. To use it, it's recommended to use your distro's
package.

If you want to use the workspaces module, first, copy the configuration files
from `/etc/xdg/waybar/` into `~/.config/waybar/`. Then, in
`~/.config/waybar/config` replace all the references to `sway/workspaces` with
`hyprland/workspaces`.

For more info regarding configuration, see
[The Waybar Wiki](https://github.com/Alexays/Waybar/wiki/Module:-Hyprland).

### How to launch

After getting everything set up, you might want to check if Waybar is configured
to your liking. To launch it, simply type `waybar` into your terminal. If you
would like Waybar to launch alongside Hyprland, you can do this by adding a line
to your Hyprland configuration that reads `exec-once=waybar`.

### Waybar popups render behind the windows

In `~/.config/waybar/config`, make sure that you have the `layer` configuration
set to `top` and not `bottom`.

### Active workspace doesn't show up

Replace `#workspaces button.focused` with `#workspaces button.active` in
`~/.config/waybar/style.css`.

### Scrolling through workspaces

Since a lot of configuration options from `sway/workspaces` are missing,
you should deduce some of them by yourself. In the case of scrolling, it should
look like this:

```json
"hyprland/workspaces": {
     "format": "{icon}",
     "on-scroll-up": "hyprctl dispatch workspace e+1",
     "on-scroll-down": "hyprctl dispatch workspace e-1"
}
```

### Window title is missing

The prefix for the window module that provides the title is `hyprland` not `wlr`.
In your Waybar config, insert this module:

```json
"modules-center": ["hyprland/window"],
```

If you are using multiple monitors, you may want to insert the following option:

```json
"hyprland/window": {
    "separate-outputs": true
},
```

## Eww

[Eww](https://github.com/elkowar/eww) (ElKowar's Wacky Widgets) is a widget system made in Rust, which lets you
create your own widgets similarly to how you can in AwesomeWM. The key difference
is that it is independent of your window manager/compositor.

In order to use [Eww](https://github.com/elkowar/eww), you first have to install
it, either using your distro's package manager, by searching `eww-wayland`, or
by manually compiling. In the latter case, you can follow the
[instructions](https://elkowar.github.io/eww).

### Configuration

After you've successfully installed Eww, you can move onto configuring it. There
are a few examples listed in the [Readme](https://github.com/elkowar/eww). It's
also highly recommended to read through the
[Configuration options](https://elkowar.github.io/eww/configuration.html).

{{< callout >}}

Read
[the Wayland section](https://elkowar.github.io/eww/configuration.html#wayland)
carefully before asking why your bar doesn't work.

{{< /callout >}}

Here are some example widgets that might be useful for Hyprland:

<details>
<summary>Workspaces widget</summary>

This widget displays a list of workspaces 1-10. Each workspace can be clicked on
to jump to it, and scrolling over the widget cycles through them. It supports
different styles for the current workspace, occupied workspaces, and empty
workspaces. It requires [bash](https://linux.die.net/man/1/bash),
[awk](https://linux.die.net/man/1/awk),
[stdbuf](https://linux.die.net/man/1/stdbuf),
[grep](https://linux.die.net/man/1/grep),
[seq](https://linux.die.net/man/1/seq),
[socat](https://linux.die.net/man/1/socat),
[jq](https://stedolan.github.io/jq/), and [Python 3](https://www.python.org/).

#### `~/.config/eww.yuck`

```lisp
...
(deflisten workspaces :initial "[]" "bash ~/.config/eww/scripts/get-workspaces")
(deflisten current_workspace :initial "1" "bash ~/.config/eww/scripts/get-active-workspace")
(defwidget workspaces []
  (eventbox :onscroll "bash ~/.config/eww/scripts/change-active-workspace {} ${current_workspace}" :class "workspaces-widget"
    (box :space-evenly true
      (label :text "${workspaces}${current_workspace}" :visible false)
      (for workspace in workspaces
        (eventbox :onclick "hyprctl dispatch workspace ${workspace.id}"
          (box :class "workspace-entry ${workspace.id == current_workspace ? "current" : ""} ${workspace.windows > 0 ? "occupied" : "empty"}"
            (label :text "${workspace.id}")
            )
          )
        )
      )
    )
  )
...
```

#### `~/.config/eww/scripts/change-active-workspace`

```sh
#!/usr/bin/env bash
function clamp {
	min=$1
	max=$2
	val=$3
	python -c "print(max($min, min($val, $max)))"
}

direction=$1
current=$2
if test "$direction" = "down"
then
	target=$(clamp 1 10 $(($current+1)))
	echo "jumping to $target"
	hyprctl dispatch workspace $target
elif test "$direction" = "up"
then
	target=$(clamp 1 10 $(($current-1)))
	echo "jumping to $target"
	hyprctl dispatch workspace $target
fi
```

#### `~/.config/eww/scripts/get-active-workspace`

```sh
#!/usr/bin/env bash

hyprctl monitors -j | jq '.[] | select(.focused) | .activeWorkspace.id'

socat -u UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock - |
  stdbuf -o0 awk -F '>>|,' -e '/^workspace>>/ {print $2}' -e '/^focusedmon>>/ {print $3}'
```

#### `~/.config/eww/scripts/get-workspaces`

```sh
#!/usr/bin/env bash

spaces (){
	WORKSPACE_WINDOWS=$(hyprctl workspaces -j | jq 'map({key: .id | tostring, value: .windows}) | from_entries')
	seq 1 10 | jq --argjson windows "${WORKSPACE_WINDOWS}" --slurp -Mc 'map(tostring) | map({id: ., windows: ($windows[.]//0)})'
}

spaces
socat -u UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock - | while read -r line; do
	spaces
done
```

</details>

<details>
<summary>Active window title widget</summary>

This widget simply displays the title of the active window. It requires
[awk](https://linux.die.net/man/1/awk),
[stdbuf](https://linux.die.net/man/1/stdbuf),
[socat](https://linux.die.net/man/1/socat), and
[jq](https://stedolan.github.io/jq/).

#### `~/.config/eww/eww.yuck`

```lisp
...
(deflisten window :initial "..." "sh ~/.config/eww/scripts/get-window-title")
(defwidget window_w []
  (box
    (label :text "${window}"
    )
  )
...
```

#### `~/.config/eww/scripts/get-window-title`

```sh
#!/bin/sh
hyprctl activewindow -j | jq --raw-output .title
socat -u UNIX-CONNECT:$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock - | stdbuf -o0 awk -F '>>|,' '/^activewindow>>/{print $3}'
```

</details>

## Hybrid

Like Waybar, [Hybrid](https://github.com/vars1ty/HybridBar) is a GTK status bar
mainly focused on wlroots compositors.

You can install it by installing `hybrid-bar` from the AUR.

### Configuration

The configuration is done through JSON, more information is available
[here](https://github.com/vars1ty/HybridBar).

### How to launch

After configuring HybridBar, you can launch it by typing `hybrid-bar` into your
terminal to try it out. It is also possible to set it to launch at start, to do
this you can add a line to your Hyprland configuration that reads
`exec-once=hybrid-bar`

#### Blur

To activate blur, set `blurls=NAMESPACE` in your Hyprland configuration, where
`NAMESPACE` is the gtk-layer-shell namespace of your HybridBar. The default
namespace is `gtk-layer-shell` and can be changed in the HybridBar configuration
at

```json
{
     "hybrid" {
          "namespace": "namespace name"
     }
}
```
