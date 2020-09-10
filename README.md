# Orbiter Plugin Utilities

This module contains partials necessary for working with plugins in other Orbiter modules.

## Partials

- `plugin`: `dict "Root" $root "Name" $name`

Get a given plugin by name. The plugin's `$name` is the name of the configuration file placed
in `/data/plugins` (see [Developing Plugins](#developing-plugins) for more), while `$root` is
the `$` passed to the highest-level template (often passed as `$.Root` to Orbiter partials, as
here).

The returned object has the following structure (as pseudo-Go code):

```go
{
    Name: string,
    Config: map[string]interface{},
    Fonts: []string,
    Shortcodes: []string,
    Csp {
        Fonts: string,
        Script: string, 
        Style: string
    },
    Partials {
        Comments: string,
        Head: string,
        Foot: string,
        Footer: string,
        Sidebar: string,
    }
}
```

    - `$.Name`: The plugin's name. Useful when plugins are retrieved using the `enabled_plugins`
      partial.
    - `$.Config`: The plugin configuration, as a map. The configuration is a result of a portion
      of the site's configuration merged with plugin defaults.
    - `$.Fonts`: A slice of all font families provided by this plugin.
    - `$.Shortcodes`: A slice of all shortcodes provided by this plugin.
    - `$.Csp`: A collections of partials that return CSP sources required by this plugin.
        - `$.Csp.Fonts`: returns strings for the `font-src` directive
        - `$.Csp.Script`: returns strings for the `script-src` directive
        - `$.Csp.Style`: returns strings for the `style-src` directive
    - `$.Partials`: Returns paths that can be used in calls to `partial` to implement the plugin.
        - `$.Partials.Comments`: The HTML for a comment provider, to be placed at the bottom
          of a content page.
        - `$.Partials.Head`: HTML to go inside the `<head>` element, for instance to load CSS.
        - `$.Partials.Foot`: HTML to put at the bottom of the `<body>` element, for instance
          to load JavaScript.
        - `$.Partials.Footer`: HTML for a site footer widget.
        - `$.Partials.Sidebar`: HTML for a site sidebar widget.

- `enabled_plugins`: `dict "Root" $root "Scope" $scope`

  Arguments: `$root` is the root variable as passed to the top-level template. `$scope` is one
  of `"site"` or `"page"`.

  Returns a slice of plugins (as returned by the `plugin` partial). The plugins included differ
  based on the value of `$scope`: both return all global plugins (sidebar, footer, and anything
  enabled in `$.Site.Params.plugins`), but either the shortcodes and comments used on the current
  page (`$scope == "page"`) or all shortcodes and comments used across the entire site
  (`$scope == "site"`).

- `head/plugins.html`

  Loads `$.Partials.Head` for every plugin on the current page. Should be included once inside
  `<head>`.

- `foot/plugins.html`

  Loads `$.Partials.Foot` for every plugin on the current page. Should be included once at the end
  of `<body>`.

## Developing Plugins

All plugins are [Hugo modules](https://gohugo.io/hugo-modules/), a concept that should be understood
before writing one. A Hugo module that registers itself with Orbiter becomes an Orbiter plugin. For
consistency, Orbiter plugins must follow certain rules and conventions.

### Plugin Registration

All Orbiter plugins must place a data file in `/data/plugins/` with a unique name that is used as
the plugins identifier (except for color scheme plugins---more on that later). The data file must
include some of the following content, including at least one partial (example uses TOML):

```toml
config_label = ""
fonts = []
shortcodes = []
[partials]
    comments = ""
    head = ""
    foot = ""
    footer = ""
    sidebar = ""
[partials.csp]
    fonts = ""
    script = ""
    style = ""
[defaults]
    # Defaults go here
```

- `config_label`: The unique configuration namespace for this plugin. As an example, for
  `config_label = "foo"`, the plugin's configuration will be retrieved from `$.Site.Params.foo`.
- `defaults`: Any default values for required configuration options, to help keep templates
  cleaner. Merged with the configuration from the plugin's namespace.
- `fonts`: All font families provided by the plugin, as strings used in a CSS `font-family`
  property.
- `shortcodes`: All shortcodes provided by the plugin, as names for which
  [`$.HasShortcode`](https://gohugo.io/templates/shortcode-templates/) will return true.
- `partials`: The entrypoints of the various partials registered by the plugin. For where they
  are included, see the [Partials](#partials) section above. For more on how to write them,
  see [Writing Plugin Partials](#writing-plugin-partials) below.
- `partials.csp`: Partials that return content security sources that must be allowed for the
  plugin to work. Should be used when a plugin uses remote fonts, CSS, and/or javascript.

#### Color Schemes Exception

Since color scheme plugins are nothing more than a `config.toml` that sets some default options
under `$.Site.Params.color`, there is no need for them to register as a plugin.

### Paths

The following list outlines a suggested layout for how yours plugin's files should appear in Hugo's
virtual filesystem. Since [module mounts](https://gohugo.io/hugo-modules/configuration/) exist, the
main concern is that files do not unintentionally clash with those provided by other plugins. Replace
`foo` with the unique identifier for your plugin.

- `assets/`
  - `css/`
    - `foo/`
      - CSS files here
  - `js/`
    - `foo/`
      - JavaScript files here
- `layouts/partials/`
  - `foo/`
    - `head.html`
    - `foot.html`
    - Other exported partials, and maybe some non-exported, internal ones
    - `csp`
      - Content-Security partials
- `static/`
  - `fonts/`
    - Folder as font name in kebab-case
      - Font files
  - `images/`
    - `foo/`
      - Images used by the plugin

(Fonts are not namespaced by plugin name in this example because the font name is considered its
own namespace)

### Writing Plugin Partials

#### Partial Parameters

All Orbiter plugin partials---both HTML-entrypoint and CSP---are passed the same context. If `$root`
is the root context provided to the top-level template and `$plugin` is the value returned from a
call to the `plugin` partial, any given partial (in this example, the `head` one) will be called:

```
{{ partial $plugin.Partials.Head (dict "Root" $root "Config" $plugin.Config) }}
```

Any site variables normally accessed through `$.Site` can instead be accessed through `$.Root.Site`.

Though it may be tempting to access your plugin's configuration directly, e.g. through
`$.Root.Site.Params.foo`, please use `$.Config` instead to keep things clean. `$.Root` is intended
for accessing any necessary global information, such as site data, *global* configuration, and
page front matter.

All non-CSP partials are expected to directly output any generated HTML, and any returned values are
ignored.

#### Error-Checking

To improve user experience, check for any user configuration errors in your templates and use Hugo's
[`errorf`](https://gohugo.io/functions/errorf/) function to indicate to the user:

- What the error is
- How to fix it, if possible
- Which content file generated the error

For example, if the error is that `$.Config.example` can only have values of `foo` or `bar`, you might
have a piece of a template that looks like this:

```
{{- $allowed := slice "foo" "bar" -}}
{{- with $.Config.example -}}
    {{- if in $allowed . -}}
        {{- /* Do stuff */ -}}
    {{- else -}}
        {{- errorf "(%s) invalid value \"%s\" for example. Valid values are %#v" $.Root.File.Path . $allowed -}}
    {{- end -}}
{{- else -}}
    {{- $valid := delimit $allowed ", " ", or " -}}
    {{- errorf "(%s) missing required configuration item example: %s" $.Root.File.Path $valid -}}
{{- end -}}
```

### Writing Content Security Partials

Each Content Security partial should `return` a slice of sources to allow in the server's `Content-Security`
policy. This allows server configuration plugins to deduplicate entries before generating the configuration.
