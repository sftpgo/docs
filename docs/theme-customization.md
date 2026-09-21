---
description: "Customize the SFTPGo web interfaces: brand color, sidebar, typography and logos through CSS variables and the branding configuration."
---

# Theme customization

The WebAdmin and WebClient are built on Bootstrap 5 with a custom theme. Colors, surfaces, borders, the sidebar and typography are all driven by CSS variables, so a rebrand is a stylesheet that redefines the variables it needs.

## The defaults file

Every customizable variable, with the value the interface ships with, is served at:

```
https://<your-server>/static/branding-defaults/theme-defaults.css
```

It is the recommended starting point: download it, keep the variables the brand requires, delete the rest. A variable that is not listed keeps its default, so the file can be trimmed to a handful of lines.

Each server serves the values of the theme it runs, so downloading the file again after an upgrade shows what moved. The copy on disk belongs to the installation and every upgrade replaces it, so keep the customized copy somewhere else.

## Applying a stylesheet

Save the customized copy under the directory configured as `static_files_path`, next to the logo and the other brand assets. A `branding` directory is the usual place, alongside the `branding-defaults` one the reference is served from. Then list it in the `extra_css` field of the binding's [branding configuration](config-file.md#branding), as a path relative to `static_files_path`:

```shell
SFTPGO_HTTPD__BINDINGS__0__BRANDING__WEB_ADMIN__EXTRA_CSS=/branding/mybrand.css
SFTPGO_HTTPD__BINDINGS__0__BRANDING__WEB_CLIENT__EXTRA_CSS=/branding/mybrand.css
```

The field is a list, so several stylesheets are a comma-separated value and are applied in the order given. The same settings can be written in the configuration file, see [branding](config-file.md#branding).

Files added there are not part of the installed package, so an upgrade leaves them untouched, while everything under `branding-defaults` is replaced.

Files listed in `extra_css` are loaded after the theme, so their declarations win. The `default_css` field replaces the theme instead, and is meant for a complete custom bundle.

The branding block is defined per binding and per interface, so the WebAdmin and the WebClient can carry different themes, and two bindings on different ports can serve two different brands.

:information_source: The configuration is read at startup: restart the service after editing it. Editing the stylesheet itself needs only a reload of the browser page, with the cache bypassed.

## Quick start: the brand color

Every primary button, active menu entry, checkbox, badge and accent follows the `primary` family:

```css
:root {
  --bs-primary:         #ff6600;
  --bs-primary-active:  #cc5200;   /* hover and pressed states */
  --bs-primary-light:   #fff1e6;   /* soft tinted background */
  --bs-primary-clarity: rgba(255, 102, 0, 0.2);
  --bs-primary-inverse: #ffffff;   /* text drawn on top of the color */
  --bs-primary-rgb:     255, 102, 0;

  --bs-link-color:       #ff6600;
  --bs-link-color-rgb:   255, 102, 0;
  --bs-link-hover-color: #cc5200;
}
```

Setting the base color alone leaves hover states and soft backgrounds on the previous hue, which reads as an unfinished rebrand. The defaults file lists the whole family together for this reason.

## The variants of a family

Colors are grouped in families: `primary` carries the brand, `success`, `info`, `warning` and `danger` carry the meaning of alerts and badges, `secondary`, `light` and `dark` are the neutrals. Each family has the same variants, and each variant answers a different question:

| Variant | What it paints |
| --- | --- |
| base, e.g. `--bs-primary` | the color itself: button and badge backgrounds, accents |
| `-active` | the hover and pressed state, usually a shade darker |
| `-light` | a soft tinted background: alerts, light buttons, expanded menu entries |
| `-clarity` | the color at low opacity, for translucent borders |
| `-inverse` | the text and icons drawn on top of the color |
| `-rgb` | the base as `r, g, b`, for the rules that compose it with an alpha |
| `-text-emphasis` | the color used as text over the page background |
| `-bg-subtle` | a very light tint, the background of alerts |
| `-border-subtle` | the matching border of those alerts |

A quick accent change needs the first six. A full rebrand sets all nine, per family and per mode, which is what the defaults file gives ready to edit.

## Surfaces and text

Outside the families, a handful of variables carry the neutral parts of the page:

| Variable | What it sets |
| --- | --- |
| `--bs-body-bg` | the page background |
| `--bs-body-color` | the default text color |
| `--bs-heading-color` | headings |
| `--bs-text-muted` | secondary text |
| `--bs-border-color` | the borders of cards, inputs and dividers |
| `--bs-gray-100` to `--bs-gray-900` | the ramp behind the `text-gray-*` and `bg-gray-*` utility classes used across the pages |

The `-rgb` companions of the first two exist for the rules that compose them with an alpha. The gray ramp is the widest lever of the set: a warmer or cooler ramp shifts every page at once.

## Light and dark

The interface follows the system preference and the in-app switch, and defines its variables twice: under `:root, [data-bs-theme=light]` and under `[data-bs-theme=dark]`.

**The selector decides which mode a value reaches.** `[data-bs-theme=light]` and `[data-bs-theme=dark]` each apply to their own mode. `:root` applies to both, whatever the mode, and wins over the shipped dark values because the custom stylesheet is loaded later:

```css
/* both modes */
:root {
  --bs-primary: #ff6600;
}

/* light mode only, dark keeps the shipped value */
[data-bs-theme=light] {
  --bs-primary: #ff6600;
}
```

A palette for each mode is therefore two blocks, one per `data-bs-theme` selector, and `:root` is the shorthand for "the same value everywhere". The base color usually needs a small luminance adjustment between the two modes, while the `-light` variant changes role completely: it is a pale tint of the brand color on a light surface and a deep tint of it on a dark one.

## The sidebar

The sidebar carries its own colors and is dark in both modes by default, so it is not affected by the brand family. A light or branded sidebar needs the text colors as well as the background, otherwise the menu entries stay tuned for a dark surface:

```css
:root {  /* a branded sidebar usually keeps one palette in both modes */
  --bs-app-sidebar-base-bg-color:                  #ffffff;
  --bs-app-sidebar-base-border-end:                1px solid #dde1e6;
  --bs-app-sidebar-base-logo-color:                #21272a;  /* name next to the logo */
  --bs-app-sidebar-base-scrollbar-color:           #c4cada;  /* scrollbar on hover */
  --bs-app-sidebar-base-menu-link-color:           #21272a;  /* menu entries */
  --bs-app-sidebar-base-menu-link-icon-color:      #697077;  /* icons, bullets, arrows */
  --bs-app-sidebar-base-menu-link-hover-color:     #32468b;
  --bs-app-sidebar-base-menu-link-active-color:    #ffffff;
  --bs-app-sidebar-base-menu-link-active-bg-color: #32468b;  /* current page */
}
```

The sidebar has no border of its own in light mode, since the shipped one is dark enough to stand out from the page: a light sidebar needs `--bs-app-sidebar-base-border-end` to keep the two apart.

The icon of the current entry keeps the icon color, so it stays readable on the background chosen for that entry.

The logo itself is an image: a logo drawn for a dark sidebar needs a second version when the sidebar becomes light. It is uploaded separately, see below.

The top header and the page background have their own variables as well: `--bs-app-header-base-bg-color`, `--bs-app-bg-color` and `--bs-app-blank-bg-color`, the last one being the background of the login and password pages.

## Typography and shape

The interface ships with Inter and falls back to the system sans-serif stack:

```css
:root {
  --bs-body-font-family: "YourBrandFont", var(--bs-font-sans-serif);
  --bs-body-font-size:   1rem;
}
```

Keeping `var(--bs-font-sans-serif)` in the list preserves the system stack if the brand font fails to load.

A font that is not installed on the client needs a `@font-face` declaration in the same stylesheet, with the font file served from the same host:

```css
@font-face {
  font-family: "YourBrandFont";
  src: url("/branding/fonts/yourbrandfont.woff2") format("woff2");
  font-weight: 400;
  font-display: swap;
}
```

:warning: Load fonts from the server that serves the interface. A font requested from a public font service sends the address of every user to a third party on each page load, and fails where outbound traffic is filtered. The font file can also be embedded in the stylesheet as a `data:` URI.

Corner radius is a family of its own, from buttons and inputs up to cards and modals. Square corners are the whole set at `0`:

```css
:root {
  --bs-border-radius:     0;
  --bs-border-radius-sm:  0;
  --bs-border-radius-lg:  0;
  --bs-border-radius-xl:  0;
  --bs-border-radius-xxl: 0;
}
```

`--bs-border-radius-pill` keeps badges and pills rounded and is usually left alone.

## Components read the theme variables

Buttons, inputs, tables and menus take their colors from the variables listed in the defaults file, not from the per-component Bootstrap variables. A primary button is drawn with `--bs-primary` and `--bs-primary-inverse`, so a rule that sets `--bs-btn-bg` or `--bs-btn-color` has no effect, and a stylesheet designed against plain Bootstrap will render differently here.

Two consequences worth checking on a mockup built elsewhere:

- the text on a colored button comes from the `-inverse` variant of that family, and stays white unless it is set. A light brand color needs `--bs-primary-inverse` set to a dark value.
- hover and pressed states come from the `-active` variant, not from a computed shade.

**Some variables belong to a component, not to the page.** Cards, modals and tables declare their own variables on themselves, and a value set on an element always beats one inherited from the root, so those have to be set on the same selector:

```css
.card  { --bs-card-bg: #fbfbfd; --bs-card-border-color: #dde1e6; }
.modal { --bs-modal-bg: #fbfbfd; }
.table { --bs-table-hover-bg: #f4f6fa; }
```

:information_source: To find the variable behind a specific element, open the interface in a browser, inspect the element and read the `var(--bs-...)` name in the computed styles. That also shows which selector declares it, which tells whether it can be set on the root.

**A rule can lose to the theme even when it loads later.** Selector specificity is compared before load order, and the theme reaches four or five classes deep in places, the sidebar menu among them: a rule of two classes never applies there. A variable answers the same need without entering that contest.

## Logos, favicon and names

The logo, the favicon, the interface name and the disclaimer link are not part of the stylesheet. They are set in the branding configuration of the binding, or from **Server Manager => Configurations => Branding** in the WebAdmin, which applies them to every instance sharing the same data provider without a restart. The logo and the favicon are uploaded as PNG images.

## Before rolling it out

The surfaces that most often stay behind:

- both modes, through the switch in the header, and the mode the browser reports when nothing is saved;
- the login page, which has its own background and card;
- a table and a modal, which carry variables of their own;
- a button in its hover, pressed and disabled state, and a focused input;
- a narrow window, where the sidebar becomes a panel over the page;
- the WebClient as well as the WebAdmin, when the stylesheet is listed for both.

## After an upgrade

Variable names are stable across releases, but an upgrade of the underlying theme can add surfaces that a custom stylesheet does not cover yet. Download the defaults file again, compare it with the previous one and test the stylesheet on a staging instance before rolling it out.

Rules that target element names or CSS classes instead of variables are tied to the current markup and are the first thing to break on an upgrade. Prefer a variable whenever one exists.

## Security note

A custom stylesheet runs in the browser of every user of the interface: it can show text, load resources and alter the layout of the pages. Write access to the directory served as `static_files_path` is therefore as sensitive as write access to the configuration, and stylesheets received from a third party should be reviewed before being installed.
