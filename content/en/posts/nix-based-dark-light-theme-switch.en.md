---
title: "Nix Based Dark Light Theme Switch"
date: 2024-02-28T17:49:40+02:00
draft: false
toc: false
images:
tags:
  - tech
  - nix
---

# Overview

Nixos configuration in a matter of theming is very useful. There are several tools to define color-theme globally and re-use defined color variables to configure any program you want, for example, [nix-colors](https://github.com/Misterio77/nix-colors) or [stylix](https://github.com/danth/stylix).

When you change a theme and switch configuration nix normally it will rebuild programm configuration file and re-start such program if it doesn't support live configuration reloading. Such a behavior can be utilized for theme-switching without the help of any other tool or custom scripts.

In my case, I'll use _Stylix_, because it provides pre-build color configurations based on base64-scheme for a lot of programs out of the box.

The final stack is:
- [home-manager](https://github.com/nix-community/home-manager)
- [stylix](https://github.com/danth/stylix)
- Mustache theme engine

# Nix configuration

## Home-manager specialisation

Home-manager provides a [specialisation feature](https://nix-community.github.io/home-manager/options.xhtml#opt-specialisation) that allows us to specify extra configuration subset based on the main one. It will generate a separate generation, that we will be able to activate manually in the future. Configuration options in specialisation and base configuration will collide, so we have to use `lib.mkDefault` in base configuration or `lib.mkForce` in specialisation.

Bellow the example of stylix configuration for `light` specialisation.

```nix
{ pkgs, lib, inputs, ... }: {

  stylix = {
    targets.gtk.enable = false;
    targets.firefox.enable = false;
    targets.swaylock.enable = false;
    targets.rofi.enable = false;
    targets.waybar.enable = false;
    targets.vscode.enable = false;
    targets.hyprland.enable = false;

    base16Scheme = lib.mkDefault "${inputs.base16-schemes}/catppuccin-macchiato.yaml";
    polarity = lib.mkDefault "dark";
    cursor = {
      package = pkgs.master.phinger-cursors;
      name = lib.mkDefault "phinger-cursors-light";
    };
  };

  specialisation.light.configuration.stylix = {
    base16Scheme = "${inputs.base16-schemes}/catppuccin-latte.yaml";
    polarity = "light";
    cursor.name = "phinger-cursors";
  };
}
```

In the same way, specialisation with the same `light` name could be specified in any other configuration file.

## Switching script

To toggle the color scheme I'll use a shell-script, that will be different for base system and specialisation.

For the base system it will get the path of the latest generation and call `/specialisation/light/activate` script. Note, that `lib.lowPrio` used to resolve collision with the spcialisation script version.

```nix
home.packages = [
  (lib.lowPrio (pkgs.writeShellApplication {
    name = "toggle-theme";
    runtimeInputs = with pkgs; [ home-manager coreutils ripgrep fish ];
    text =
      ''
        "$(home-manager generations | head -1 | rg -o '/[^ ]*')"/specialisation/light/activate
        fish --command 'set -U _reload_theme (date +%s)'
      '';
  }))
];
```

For the specialisation it will get the path of the 2nd latest generation, because in the moment of call the latest generation will be related to `light` system version. And call `/activate` script.

```nix
specialisation.light.configuration = {
  home.packages = [
    (pkgs.writeShellApplication {
      name = "toggle-theme";
      runtimeInputs = with pkgs; [ home-manager coreutils ripgrep fish ];
      text =
        ''
          "$(home-manager generations | head -2 | tail -1 | rg -o '/[^ ]*')"/activate
          fish --command 'set -U _reload_theme (date +%s)'
        '';
    })
  ];
```

### Fish shell reload

Fish shell redefines all the colors and ignores terminal ones, so when we switch themes we must re-login into each shell, which is not handy at all and is not possible to be resolved with pure nix. As a solution, we can use a function that will reload the theme and be triggered by setting [universal variable](https://fishshell.com/docs/current/language.html#universal-variables).


To avoid reoading of whole configuration, which may lead to unexpected behaviour, this scripp will find 2 lines responsible for theme loading and switching and source them.

```nix
programs.fish = {
  enable = true;
  interactiveShellInit = ''
    function reload-theme --on-variable _reload_theme
      # Apply theme on theme toggle trigger
      ${pkgs.ripgrep}/bin/rg '^\s*source /nix/store/.+base16-.+-.+fish$' ~/.config/fish/config.fish -A 1 -N --trim | source
    end
  '';
};
```

In switching script you may notice `fish --command 'set -U _reload_theme (date +%s)'` this line triggers fish's `reload-theme` function.

## Mustache for custom themes

For configuration it is very handy to use template engide, mustache becomes very useful, because it supported by Stylix out of the box. For example `rofi` configuration can be looked like this.

```nix
programs.rofi = {
  enable = true;
  font = "${config.stylix.fonts.serif.name} 20";
  theme = (config.lib.stylix.colors {
    template = builtins.readFile ./rounded.rasi.mustache;
    extension = "rasi";
  });
}
```

And the corresponding theme file like this.

```mustache
* {
    bg0:    #{{base00}}F0;
    bg1:    #{{base01}};
    bg2:    #{{base02}}80;
    bg3:    #{{base03}}F2;
    fg0:    #{{base04}};
    fg1:    #{{base05}};
    fg2:    #{{base06}};
    fg3:    #{{base07}};
    fg4:    #{{base00}};
}

...
```

# Reference
- [Post with original toggle-script from Janik in the nix discourse](https://discourse.nixos.org/t/home-manager-toggle-between-themes/32907)