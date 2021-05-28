# A guide to Helm

<br />

This project contains helm chart templates from the folders accessible from the root directory.

Just clone this repo, and within the folder of a choice, do a `helm install <name-of-your-chart> helm/` to see the application deployed in your kubernetes cluster.

For summarized notes on how helm works, refer to the sections below.

<br />


## Getting Started

To test template rendering, but not actually installing anything, use the `dry-run` option:

`helm install --debug --dry-run mychart ./mychart`

<br />

## Built-in Objects

Objects can be passed into a template from the template engine.

For example, we can use `{{ .Release.Name }}` to insert the name of a release into a template. Other variables include:

- Release.Name
- Release.Namespace
- Release.IsUpgrade
- Release.IsInstall
- Release.Revision

We can also use `Values` from the values.yaml file, or `Chart` from the Chart.yaml file.

For `Template`:
> `Template.Name` is a namespaced file path to the current template (e.g. `mychart/templates/mytemplate.yaml`)


<br />

## Template Functions & Pipelines

Template functions follow the syntax `functionName arg1 arg2 . . .`.

For example: `{{ quote .Values.favorite.food }}` calls the `quote` function and passes it a single argument.

Pipelines draws from a concept from UNIX, and is a tool for chaining together a series of template commands for transformations.

For example: 
> `{{ .Values.favorite.food | upper | quote }}` will result in "PIZZA".
>
> `{{ .Values.favorite.drink | default "tea" | quote }}` will result in "tea" if the original template value is omitted.


<br />

## Flow Control

`{{-` indicates that the whitespace should be chomped left, while `-}}` indicates that whitespace to the right should be consumed.

```yaml
{{- if eq .Values.favorite.drink "coffee" }}
mug: true
{{- end }}
```

To change scope, use `with`

```yaml
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

To loop, use `range`

Say we have a values.yaml with the following

```yaml
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

We have the following code in ConfigMap

```yaml
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
  {{- range .Values.pizzaToppings }}
  - {{ . | title | quote }}
  {{- end }}    
```

<br />

## Variables

In Helm templates, a variable is a named reference to another object. It follows this syntax `$name`. 
Variables are assigned with a special assignment operator `:=`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

Variables are useful in `range` loops.

```yaml
toppings: |-
  {{- range $index, $topping := .Values.pizzaToppings }}
    {{ $index }}: {{ $topping }}
  {{- end }}    
```

<br />

## Named Templates

Naming Conventions

- Most files in `templates/` are treated as if they contain Kubernetes manifests
- The NOTES.txt is one execption
- But files whose file name begin with an underscore `_` are assumed to not have a manifest inside. for Example `_helper.tpl`

Declaring and using templates with `define` and `template`

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

will result in the following template

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: running-panda-configmap
  labels:
    generator: helm
    date: 2016-11-02
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "pizza"
```

Conventionally, Helm charts put these templates inside of a partials file, usually `_helpers.tpl`. 
Removing the `{{- define ... }}` section will still allow it to be accessed in configmap.yaml.

Next, use the `indent` feature with `include` to indent template correctly.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
```

<br />

## Accessing Files Inside Templates

Helm provides access to external files through the `.Files` object.
We need to be careful with the size however, as files are bundled and Charts must be smaller than 1M because of storage limitations of K8s objects.
Files in `templates/` also cannot be accessed, as well as files excluded using `.helmignore`.

Example of including files:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  {{- $files := .Files }}
  {{- range tuple "config1.toml" "config2.toml" "config3.toml" }}
  {{ . }}: |-
        {{ $files.Get . }}
  {{- end }}
```

Here, the `tuple` function creates a list of files that we loop through. 
Then we print each file name `{{ . }}` followed by the contents of the file `{{$files.Get .}}`.

The result is:

```yaml
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: quieting-giraf-configmap
data:
  config1.toml: |-
        message = Hello from config 1

  config2.toml: |-
        message = This is config 2

  config3.toml: |-
        message = Goodbye from config 3
```

It is very common to want to place file content into both ConfigMaps and Secrets, for mounting into pods at run time.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: conf
data:
{{ (.Files.Glob "foo/*").AsConfig | indent 2 }}
---
apiVersion: v1
kind: Secret
metadata:
  name: very-secret
type: Opaque
data:
{{ (.Files.Glob "bar/*").AsSecrets | indent 2 }}
```

We can also do a `{{ .Files.Get "config1.toml" | b64enc }}` to encode it.


<br />

## Creating a NOTES.txt

At the end of a `helm install` or `helm upgrade`, Helm can print helpful information for users through this file.


<br />

## Subcharts and global values

1. A subchart is considered "stand-alone". It can never explicitly depend on its parent chart, and thus cannot access the values of its parent.
2. A parent chart can override values for subcharts.
3. Helm has a concept of *global values* that can be accessed by all charts.


<br />

## Debugging templates

- `helm lint` to verify chart follows best practices
- `helm install --dry-run --debug` or `helm template --debug`
- `helm get manifest` to see what templates are installed on the server

<br />
### Appendix

The word for an empty value is `null` not `nil`.

In some cases, force type inference using YAML node tags

```yaml
age: !!str 21
port: !!int "80"
```

**Strings**

All inline styles must be on one line.

```yaml
way1: bare words
way2: "double-quoted strings"
way3: 'single-quoted strings'
```

- Bare words are unquoted, and are not escaped. For this reason, you have to be careful what characters you use.
- Double-quoted strings can have specific characters escaped with \. For example `"\"Hello\", she said"`. You can escape line breaks with \n.
- Single-quoted strings are "literal" strings, and do not use the \ to escape characters. The only escape sequence is '', which is decoded as a single '.



