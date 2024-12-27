# Tips and tricks

## Adjusting the brightness and contrast of a background image

If you want to use a background image for your desktop but find it too bright or distracting, you can use the `imagemagick` package to dim the image, or adjust its brightness and contrast to suit your preference.

Here's an example Nix expression that takes an input image, applies a brightness/contrast adjustment to it, and saves the result as a new image file:

```nix
{ pkgs, ... }:

let
  inputImage = ./path/to/image.jpg;
  brightness = -30;
  contrast = 0;
  fillColor = "black"
in
{
  stylix.image = pkgs.runCommand "dimmed-background.png" { } ''
    ${pkgs.imagemagick}/bin/convert "${inputImage}" -brightness-contrast ${brightness},${contrast} -fill ${fillColor} $out
  '';
}
```

## Dynamic wallpaper generation based on selected theme

With imagemagick, you can also dynamically generate wallpapers based on the selected theme.
Similarly, you can use a template image and repaint it for the current theme.

```nix
{ pkgs, ... }:

let
  theme = "${pkgs.base16-schemes}/share/themes/catppuccin-latte.yaml";
  wallpaper = pkgs.runCommand "image.png" {} ''
        COLOR=$(${pkgs.yq}/bin/yq -r .base00 ${theme})
        COLOR="#"$COLOR
        ${pkgs.imagemagick}/bin/magick -size 1920x1080 xc:$COLOR $out
  '';
in {
  stylix = {
    image = wallpaper;
    base16Scheme = theme;
  };
}
```

Which is neatly implemented as a single function in `lib.stylix.pixel`:

```nix
{ pkgs, config, ... }:

{
  stylix = {
    image = config.lib.stylix.pixel "base0A";
    base16Scheme = "${pkgs.base16-schemes}/share/themes/catppuccin-latte.yaml";
  };
}
```

## Completely disabling some stylix targets

Nixpkgs module system sometimes works in non-intuitive ways, e.g. parts
of the configuration guarded by `lib.mkIf` are still being descended
into. This means that every **loaded** (and not enabled) module must
be compatible with others - in the sense that **every** option that is
mentioned in the disabled parts of the configuration still needs to be
defined somewhere.

Sometimes that can be a problem, when your particular configuration
diverges enough from what stylix expects. In that case you can try
stubbing all the missing options in your configuration.

Or in a much clearer fashion you can just disable offending stylix targets
by adding the following `disableModules` line next to importing stylix
itself:

```nix
imports = [ flake.inputs.stylix.nixosModules.stylix ];
disabledModules = [ "${flake.inputs.stylix}/modules/<some-module>/nixos.nix" ];
```

## Automatic theme switcher

By leveraging [Nix Specialisations](https://nixos.wiki/wiki/Specialisation) and [darkman](https://gitlab.com/WhyNotHugo/darkman), you can set up dark-mode and light-mode transitions.

You can have Stylix as a NixOS module or a Home Manager module, but the specialisation must be added to your Home Manager configuration to not require sudo privileges when activating.

### Stylix

Keep your previous Stylix config, it will be loaded on system boot. Shortly after, darkman will start up and switch to the right theme.

```nix
# stylix-specialisation.nix in Home Manager config
{
  pkgs,
  ...
}:
{
  specialisation = {
    # Theme-specific settings
    light.configuration.stylix = {
      image = ./PATH/TO/LIGHT_BG;
      base16Scheme = "LIGHT_THEME";
      # ...
    };

    dark.configuration.stylix = {
      image = ./PATH/TO/DARK_BG;
      base16Scheme = "DARK_THEME";
      # ...
    };
  };
}
```

### Darkman

This will configure darkman to activate the `light` or `dark` specialisation of the latest Home Manager generation on theme switch.

```nix
# darkman.nix in Home Manager config
{
  pkgs,
  ...
}:
{
  services.darkman =
    let
      find-hm-generation =
        let
          home-manager = "${pkgs.home-manager}/bin/home-manager";
          grep = "${pkgs.toybox}/bin/grep";
          head = "${pkgs.toybox}/bin/head";
          find = "${pkgs.toybox}/bin/find";
        in
        # We need to find the current HM generation to get the runtime activation script
        ''
          for line in $(${home-manager} generations | ${grep} -o '/.*')
          do
            res=$(${find} $line | ${grep} specialisation | ${head} -1)
            output=$?
            if [[ $output -eq 0 ]] && [[ $res != "" ]]; then
                echo $res
                exit
            fi
          done
        '';

      # Some other services may need to be restarted here
      switch-theme-script = theme: ''
        $(${find-hm-generation})/${theme}/activate
      '';
    in
    {
      enable = true;
      darkModeScripts = {
        activate = switch-theme-script "dark";
      };
      lightModeScripts = {
        activate = switch-theme-script "light";
      };
      settings = {
        # https://darkman.whynothugo.nl/#CONFIGURATION
        lat = 48.86; # Setup your location here to switch on day/night cycle
        lng = 2.35;
        dbusserver = true;
      };
    };
}
```
