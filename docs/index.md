# gomplate

A [Go template](https://golang.org/pkg/text/template/)-based CLI tool. `gomplate` can be used as an alternative to
[`envsubst`](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html) but also supports
additional template datasources such as: JSON, YAML, AWS EC2 metadata, and
[Hashicorp Vault](https://https://www.vaultproject.io/) secrets.

I really like `envsubst` for use as a super-minimalist template processor. But its simplicity is also its biggest flaw: it's all-or-nothing with shell-like variables.

Gomplate is an alternative that will let you process templates which also include shell-like variables. Also there are some useful built-in functions that can be used to make templates even more expressive.

!-- TOC depthFrom:2 depthTo:4 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Installing](#installing)
	- [macOS with homebrew](#macos-with-homebrew)
	- [Alpine Linux](#alpine-linux)
	- [use with Docker](#use-with-docker)
	- [manual install](#manual-install)
	- [install with `go get`](#install-with-go-get)
- [Usage](#usage)
	- [Commandline Arguments](#commandline-arguments)
		- [`--file`/`-f`, `--in`/`-i`, and `--out`/`-o`](#-file-f-in-i-and-out-o)
		- [`--datasource`/`-d`](#-datasource-d)
		- [Overriding the template delimiters](#overriding-the-template-delimiters)
- [Syntax](#syntax)
	- [About `.Env`](#about-env)
	- [Built-in functions](#built-in-functions)
		- [`contains`](#contains)
		- [`getenv`](#getenv)
		- [`hasPrefix`](#hasprefix)
		- [`hasSuffix`](#hassuffix)
		- [`bool`](#bool)
		- [`slice`](#slice)
		- [`split`](#split)
		- [`splitN`](#splitn)
		- [`replaceAll`](#replaceall)
		- [`title`](#title)
		- [`toLower`](#tolower)
		- [`toUpper`](#toupper)
		- [`trim`](#trim)
		- [`urlParse`](#urlparse)
		- [`has`](#has)
		- [`join`](#join)
		- [`indent`](#indent)
		- [`json`](#json)
		- [`jsonArray`](#jsonarray)
		- [`yaml`](#yaml)
		- [`yamlArray`](#yamlarray)
		- [`toJSON`](#tojson)
		- [`toJSONPretty`](#tojsonpretty)
		- [`toYAML`](#toyaml)
		- [`datasource`](#datasource)
		- [`datasourceExists`](#datasourceexists)
		- [`ds`](#ds)
		- [`ec2meta`](#ec2meta)
		- [`ec2dynamic`](#ec2dynamic)
		- [`ec2region`](#ec2region)
		- [`ec2tag`](#ec2tag)
	- [Some more complex examples](#some-more-complex-examples)
		- [Variable assignment and `if`/`else`](#variable-assignment-and-ifelse)
- [License](#license)

<!-- /TOC -->

## Installing

### macOS with homebrew

The simplest method for macOS is to use homebrew:

```console
$ brew tap hairyhenderson/tap
$ brew install gomplate
...
```

### Alpine Linux

Currently, `gomplate` is available in the `community` repository for the `edge` release.

```console
$ echo "http://dl-cdn.alpinelinux.org/alpine/edge/community/" >> /etc/apk/repositories
$ apk update
$ apk add gomplate
...
```

_Note: the Alpine version of gomplate may lag behind the latest release of gomplate._

### use with Docker

A simple way to get started is with the Docker image.

```console
$ docker run hairyhenderson/gomplate --version
```

Of course, there are some drawbacks - any files to be used for [datasources][]
must be mounted and any environment variables to be used must be passed through:

```console
$ echo 'My voice is my {{.Env.THING}}. {{(datasource "vault").value}}' \
  | docker run -e THING=passport -v /home/me/.vault-token:/root/.vault-token hairyhenderson/gomplate -d vault=vault:///secret/sneakers
My voice is my passport. Verify me.
```

It can be pretty awkward to always type `docker run hairyhenderson/gomplate`,
so this can be made simpler with a shell alias:

```console
$ alias gomplate=docker run hairyhenderson/gomplate
$ gomplate --version
gomplate version 1.2.3
```

### manual install

1. Get the latest `gomplate` for your platform from the [releases](https://github.com/hairyhenderson/gomplate/releases) page
2. Store the downloaded binary somewhere in your path as `gomplate` (or `gomplate.exe`
  on Windows)
3. Make sure it's executable (on Linux/macOS)
3. Test it out with `gomplate --help`!

In other words:

```console
$ curl -o /usr/local/bin/gomplate -sSL https://github.com/hairyhenderson/gomplate/releases/download/<version>/gomplate_<os>-<arch>
$ chmod 755 /usr/local/bin/gomplate
$ gomplate --help
...
```

### install with `go get`

If you're a Go user already, sometimes it's faster to just use `go get` to install `gomplate`:

```console
$ go get github.com/hairyhenderson/gomplate
$ gomplate --help
...
```

_Please report any bugs found in the [issue tracker](https://github.com/hairyhenderson/gomplate/issues/)._

## Usage

The usual and most basic usage of `gomplate` is to just replace environment variables. All environment variables are available by referencing `.Env` (or `getenv`) in the template.

The template is read from standard in, and written to standard out.

Use it like this:

```console
$ echo "Hello, {{.Env.USER}}" | gomplate
Hello, hairyhenderson
```

### Commandline Arguments

#### `--file`/`-f`, `--in`/`-i`, and `--out`/`-o`

By default, `gomplate` will read from `Stdin` and write to `Stdout`. This behaviour can be changed.

- Use `--file`/`-f` to use a specific input template file. The special value `-` means `Stdin`.
- Use `--out`/`-o` to save output to file. The special value `-` means `Stdout`.
- Use `--in`/`-i` if you want to set the input template right on the commandline. This overrides `--file`. Because of shell command line lengths, it's probably not a good idea to use a very long value with this argument.

##### Multiple inputs

You can specify multiple `--file` and `--out` arguments. The same number of each much be given. This allows `gomplate` to process multiple templates _slightly_ faster than invoking `gomplate` multiple times in a row.

##### `--input-dir` and `--output-dir`

For processing multiple templates in a directory you can use `--input-dir` and `--output-dir` together. In this case all files in input directory will be processed as templates and the resulting files stored in `--output-dir`. The output directory will be created if it does not exist and the directory structure of the input directory will be preserved.  

Example:

```bash
 # Process all files in directory "templates" with the datasource given
 # and store the files with the same directory structure in "config"
gomplate --input-dir=templates --output-dir=config --datasource config=config.yaml
```

#### `--datasource`/`-d`

Add a data source in `name=URL` form. Specify multiple times to add multiple sources. The data can then be used by the [`datasource`](#datasource) function.

A few different forms are valid:
- `mydata=file:///tmp/my/file.json`
  - Create a data source named `mydata` which is read from `/tmp/my/file.json`. This form is valid for any file in any path.
- `mydata=file.json`
  - Create a data source named `mydata` which is read from `file.json` (in the current working directory). This form is only valid for files in the current directory.
- `mydata.json`
  - This form infers the name from the file name (without extension). Only valid for files in the current directory.

#### Overriding the template delimiters

Sometimes it's necessary to override the default template delimiters (`{{`/`}}`).
Use `--left-delim`/`--right-delim` or set `$GOMPLATE_LEFT_DELIM`/`$GOMPLATE_RIGHT_DELIM`.

## Syntax

### About `.Env`

You can easily access environment variables with `.Env`, but there's a catch:
if you try to reference an environment variable that doesn't exist, parsing
will fail and `gomplate` will exit with an error condition.

Sometimes, this behaviour is desired; if the output is unusable without certain strings, this is a sure way to know that variables are missing!

If you want different behaviour, try `getenv` (below).

### Built-in functions

In addition to all of the functions and operators that the [Go template](https://golang.org/pkg/text/template/)
language provides (`if`, `else`, `eq`, `and`, `or`, `range`, etc...), there are
some additional functions baked in to `gomplate`:

#### `contains`

Contains reports whether the second string is contained within the first. Equivalent to
[strings.Contains](https://golang.org/pkg/strings#Contains)

##### Example

_`input.tmpl`:_
```
{{if contains .Env.FOO "f"}}yes{{else}}no{{end}}
```

```console
$ FOO=foo gomplate < input.tmpl
yes
$ FOO=bar gomplate < input.tmpl
no
```

#### `getenv`

Exposes the [os.Getenv](https://golang.org/pkg/os/#Getenv) function.

This is a more forgiving alternative to using `.Env`, since missing keys will
return an empty string.

An optional default value can be given as well.

##### Example

```console
$ gomplate -i 'Hello, {{getenv "USER"}}'
Hello, hairyhenderson
$ gomplate -i 'Hey, {{getenv "FIRSTNAME" "you"}}!'
Hey, you!
```

#### `hasPrefix`

Tests whether the string begins with a certain substring. Equivalent to
[strings.HasPrefix](https://golang.org/pkg/strings#HasPrefix)

##### Example

_`input.tmpl`:_
```
{{if hasPrefix .Env.URL "https"}}foo{{else}}bar{{end}}
```

```console
$ URL=http://example.com gomplate < input.tmpl
bar
$ URL=https://example.com gomplate < input.tmpl
foo
```

#### `hasSuffix`

Tests whether the string ends with a certain substring. Equivalent to
[strings.HasSuffix](https://golang.org/pkg/strings#HasSuffix)

##### Example

_`input.tmpl`:_
```
{{.Env.URL}}{{if not (hasSuffix .Env.URL ":80")}}:80{{end}}
```

```console
$ URL=http://example.com gomplate < input.tmpl
http://example.com:80
```

#### `bool`

Converts a true-ish string to a boolean. Can be used to simplify conditional statements based on environment variables or other text input.

##### Example

_`input.tmpl`:_
```
{{if bool (getenv "FOO")}}foo{{else}}bar{{end}}
```

```console
$ gomplate < input.tmpl
bar
$ FOO=true gomplate < input.tmpl
foo
```

#### `slice`

Creates a slice. Useful when needing to `range` over a bunch of variables.

##### Example

_`input.tmpl`:_
```
{{range slice "Bart" "Lisa" "Maggie"}}
Hello, {{.}}
{{- end}}
```

```console
$ gomplate < input.tmpl
Hello, Bart
Hello, Lisa
Hello, Maggie
```

#### `split`

Creates a slice by splitting a string on a given delimiter. Equivalent to
[strings.Split](https://golang.org/pkg/strings#Split)

##### Example

```console
$ gomplate -i '{{range split "Bart,Lisa,Maggie" ","}}Hello, {{.}}{{end}}'
Hello, Bart
Hello, Lisa
Hello, Maggie
```

#### `splitN`

Creates a slice by splitting a string on a given delimiter. The count determines
the number of substrings to return. Equivalent to [strings.SplitN](https://golang.org/pkg/strings#SplitN)

##### Example

```console
$ gomplate -i '{{ range splitN "foo:bar:baz" ":" 2 }}{{.}}{{end}}'
foo
bar:baz
```
#### `replaceAll`

Replaces all occurrences of a given string with another.

##### Example

```console
$ gomplate -i '{{ replaceAll "." "-" "172.21.1.42" }}'
172-21-1-42
```

##### Example (with pipeline)

```console
$ gomplate -i '{{ "172.21.1.42" | replaceAll "." "-" }}'
172-21-1-42
```

#### `title`

Convert to title-case. Equivalent to [strings.Title](https://golang.org/pkg/strings/#Title)

##### Example

```console
$ echo '{{title "hello, world!"}}' | gomplate
Hello, World!
```

#### `toLower`

Convert to lower-case. Equivalent to [strings.ToLower](https://golang.org/pkg/strings/#ToLower)

##### Example

```console
$ echo '{{toLower "HELLO, WORLD!"}}' | gomplate
hello, world!
```

#### `toUpper`

Convert to upper-case. Equivalent to [strings.ToUpper](https://golang.org/pkg/strings/#ToUpper)

##### Example

```console
$ echo '{{toUpper "hello, world!"}}' | gomplate
HELLO, WORLD!
```

#### `trim`

Trims a string by removing the given characters from the beginning and end of
the string. Equivalent to [strings.Trim](https://golang.org/pkg/strings/#Trim)

##### Example

_`input.tmpl`:_
```
Hello, {{trim .Env.FOO " "}}!
```

```console
$ FOO="  world " | gomplate < input.tmpl
Hello, world!
```

#### `urlParse`

Parses a string as a URL for later use. Equivalent to [url.Parse](https://golang.org/pkg/net/url/#Parse)

##### Example

_`input.tmpl`:_
```
{{ $u := urlParse "https://example.com:443/foo/bar" }}
The scheme is {{ $u.Scheme }}
The host is {{ $u.Host }}
The path is {{ $u.Path }}
```

```console
$ gomplate < input.tmpl
The scheme is https
The host is example.com:443
The path is /foo/bar
```

#### `has`

Has reports whether or not a given object has a property with the given key. Can be used with `if` to prevent the template from trying to access a non-existent property in an object.

##### Example

_Let's say we're using a Vault datasource..._

_`input.tmpl`:_
```
{{ $secret := datasource "vault" "mysecret" -}}
The secret is '
{{- if (has $secret "value") }}
{{- $secret.value }}
{{- else }}
{{- $secret | toYAML }}
{{- end }}'
```

If the `secret/foo/mysecret` secret in Vault has a property named `value` set to `supersecret`:

```console
$ gomplate -d vault:///secret/foo < input.tmpl
The secret is 'supersecret'
```

On the other hand, if there is no `value` property:

```console
$ gomplate -d vault:///secret/foo < input.tmpl
The secret is 'foo: bar'
```

#### `join`

Concatenates the elements of an array to create a string. The separator string sep is placed between elements in the resulting string.

##### Example

_`input.tmpl`_
```
{{ $a := `[1, 2, 3]` | jsonArray }}
{{ join $a "-" }}
```

```console
$ gomplate -f input.tmpl
1-2-3
```

#### `indent`

Indents a given string with the given indentation pattern. If the input string has multiple lines, each line will be indented.

##### Example

This function can be especially useful when adding YAML snippets into other YAML documents, where indentation is important:

_`input.tmpl`:_
```
foo:
{{ `{"bar": {"baz": 2}}` | json | toYAML | indent "  " }}
```

```console
$ gomplate -f input.tmpl
foo:
  bar:
    baz: 2
```

#### `json`

Converts a JSON string into an object. Only works for JSON Objects (not Arrays or other valid JSON types). This can be used to access properties of JSON objects.

##### Example

_`input.tmpl`:_
```
Hello {{ (getenv "FOO" | json).hello }}
```

```console
$ export FOO='{"hello":"world"}'
$ gomplate < input.tmpl
Hello world
```

#### `jsonArray`

Converts a JSON string into a slice. Only works for JSON Arrays.

##### Example

_`input.tmpl`:_
```
Hello {{ index (getenv "FOO" | jsonArray) 1 }}
```

```console
$ export FOO='[ "you", "world" ]'
$ gomplate < input.tmpl
Hello world
```

#### `yaml`

Converts a YAML string into an object. Only works for YAML Objects (not Arrays or other valid YAML types). This can be used to access properties of YAML objects.

##### Example

_`input.tmpl`:_
```
Hello {{ (getenv "FOO" | yaml).hello }}
```

```console
$ export FOO='hello: world'
$ gomplate < input.tmpl
Hello world
```

#### `yamlArray`

Converts a YAML string into a slice. Only works for YAML Arrays.

##### Example

_`input.tmpl`:_
```
Hello {{ index (getenv "FOO" | yamlArray) 1 }}
```

```console
$ export FOO='[ "you", "world" ]'
$ gomplate < input.tmpl
Hello world
```

#### `toJSON`

Converts an object to a JSON document. Input objects may be the result of `json`, `yaml`, `jsonArray`, or `yamlArray` functions, or they could be provided by a `datasource`.

##### Example

_This is obviously contrived - `json` is used to create an object._

_`input.tmpl`:_
```
{{ (`{"foo":{"hello":"world"}}` | json).foo | toJSON }}
```

```console
$ gomplate < input.tmpl
{"hello":"world"}
```

#### `toJSONPretty`

Converts an object to a pretty-printed (or _indented_) JSON document. Input objects may be the result of `json`, `yaml`, `jsonArray`, or `yamlArray` functions, or they could be provided by a `datasource`.

The indent string must be provided as an argument.

##### Example

_`input.tmpl`:_
```
{{ `{"hello":"world"}` | json | toJSONPretty "  " }}
```

```console
$ gomplate < input.tmpl
{
  "hello": "world"
}
```

#### `toYAML`

Converts an object to a YAML document. Input objects may be the result of `json`, `yaml`, `jsonArray`, or `yamlArray` functions, or they could be provided by a `datasource`.

##### Example

_This is obviously contrived - `json` is used to create an object._

_`input.tmpl`:_
```
{{ (`{"foo":{"hello":"world"}}` | json).foo | toYAML }}
```

```console
$ gomplate < input.tmpl
hello: world

```

#### `datasource`

Parses a given datasource (provided by the [`--datasource/-d`](#--datasource-d) argument).

Currently, `file://`, `http://`, `https://`, and `vault://` URLs are supported.

Currently-supported formats are JSON and YAML.

##### Examples

###### Basic usage

_`person.json`:_
```json
{
  "name": "Dave"
}
```

_`input.tmpl`:_
```
Hello {{ (datasource "person").name }}
```

```console
$ gomplate -d person.json < input.tmpl
Hello Dave
```

###### Usage with HTTP data

```console
$ echo 'Hello there, {{(datasource "foo").headers.Host}}...' | gomplate -d foo=https://httpbin.org/get
Hello there, httpbin.org...
```

Additional headers can be provided with the `--datasource-header`/`-H` option:

```console
$ gomplate -d foo=https://httpbin.org/get -H 'foo=Foo: bar' -i '{{(datasource "foo").headers.Foo}}'
bar
```

###### Usage with Vault data

The special `vault://` URL scheme can be used to retrieve data from [Hashicorp
Vault](https://vaultproject.io). To use this, you must put the Vault server's
URL in the `$VAULT_ADDR` environment variable.

This table describes the currently-supported authentication mechanisms and how to use them, in order of precedence:

| auth backend | configuration |
| ---: |--|
| [`approle`](https://www.vaultproject.io/docs/auth/approle.html) | Environment variables `$VAULT_ROLE_ID` and `$VAULT_SECRET_ID` must be set to the appropriate values.<br/> If the backend is mounted to a different location, set `$VAULT_AUTH_APPROLE_MOUNT`. |
| [`app-id`](https://www.vaultproject.io/docs/auth/app-id.html) | Environment variables `$VAULT_APP_ID` and `$VAULT_USER_ID` must be set to the appropriate values.<br/> If the backend is mounted to a different location, set `$VAULT_AUTH_APP_ID_MOUNT`. |
| [`github`](https://www.vaultproject.io/docs/auth/github.html) | Environment variable `$VAULT_AUTH_GITHUB_TOKEN` must be set to an appropriate value.<br/> If the backend is mounted to a different location, set `$VAULT_AUTH_GITHUB_MOUNT`. |
| [`userpass`](https://www.vaultproject.io/docs/auth/userpass.html) | Environment variables `$VAULT_AUTH_USERNAME` and `$VAULT_AUTH_PASSWORD` must be set to the appropriate values.<br/> If the backend is mounted to a different location, set `$VAULT_AUTH_USERPASS_MOUNT`. |
| [`token`](https://www.vaultproject.io/docs/auth/token.html) | Determined from either the `$VAULT_TOKEN` environment variable, or read from the file `~/.vault-token` |

_**Note:**_ The secret values listed in the above table can either be set in environment
variables or provided in files. This can increase security when using
[Docker Swarm Secrets](https://docs.docker.com/engine/swarm/secrets/), for example.
To use files, specify the filename by appending `_FILE` to the environment variable,
(i.e. `VAULT_USER_ID_FILE`). If the non-file variable is set, this will override
any `_FILE` variable and the secret file will be ignored.

To use a Vault datasource with a single secret, just use a URL of
`vault:///secret/mysecret`. Note the 3 `/`s - the host portion of the URL is left
empty.

```console
$ echo 'My voice is my passport. {{(datasource "vault").value}}' \
  | gomplate -d vault=vault:///secret/sneakers
My voice is my passport. Verify me.
```

You can also specify the secret path in the template by using a URL of `vault://`
(or `vault:///`, or `vault:`):
```console
$ echo 'My voice is my passport. {{(datasource "vault" "secret/sneakers").value}}' \
  | gomplate -d vault=vault://
My voice is my passport. Verify me.
```

And the two can be mixed to scope secrets to a specific namespace:

```console
$ echo 'db_password={{(datasource "vault" "db/pass").value}}' \
  | gomplate -d vault=vault:///secret/production
db_password=prodsecret
```

#### `datasourceExists`

Tests whether or not a given datasource was defined on the commandline (with the
[`--datasource/-d`](#--datasource-d) argument). This is intended mainly to allow
a template to be rendered differently whether or not a given datasource was
defined.

Note: this does _not_ verify if the datasource is reachable.

Useful when used in an `if`/`else` block

```console
$ echo '{{if (datasourceExists "test")}}{{datasource "test"}}{{else}}no worries{{end}}' | gomplate
no worries
```

#### `ds`

Alias to [`datasource`](#datasource)

#### `ec2meta`

Queries AWS [EC2 Instance Metadata](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) for information. This only retrieves data in the `meta-data` path -- for data in the `dynamic` path use `ec2dynamic`.

This only works when running `gomplate` on an EC2 instance. If the EC2 instance metadata API isn't available, the tool will timeout and fail.

##### Example

```console
$ echo '{{ec2meta "instance-id"}}' | gomplate
i-12345678
```

#### `ec2dynamic`

Queries AWS [EC2 Instance Dynamic Metadata](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) for information. This only retrieves data in the `dynamic` path -- for data in the `meta-data` path use `ec2meta`.

This only works when running `gomplate` on an EC2 instance. If the EC2 instance metadata API isn't available, the tool will timeout and fail.

##### Example

```console
$ echo '{{ (ec2dynamic "instance-identity/document" | json).region }}' | ./gomplate
us-east-1
```

#### `ec2region`

Queries AWS to get the region. An optional default can be provided, or returns
`unknown` if it can't be determined for some reason.

##### Example

_In EC2_
```console
$ echo '{{ ec2region }}' | ./gomplate
us-east-1
```
_Not in EC2_
```console
$ echo '{{ ec2region }}' | ./gomplate
unknown
$ echo '{{ ec2region "foo" }}' | ./gomplate
foo
```

#### `ec2tag`

Queries the AWS EC2 API to find the value of the given [user-defined tag](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html). An optional default
can be provided.

##### Example

```console
$ echo 'This server is in the {{ ec2tag "Account" }} account.' | ./gomplate
foo
$ echo 'I am a {{ ec2tag "classification" "meat popsicle" }}.' | ./gomplate
I am a meat popsicle.
```

### Some more complex examples

#### Variable assignment and `if`/`else`

_`input.tmpl`:_
```
{{ $u := getenv "USER" }}
{{ if eq $u "root" -}}
You are root!
{{- else -}}
You are not root :(
{{- end}}
```

```console
$ gomplate < input.tmpl
You are not root :(
$ sudo gomplate < input.tmpl
You are root!
```

## License

[The MIT License](http://opensource.org/licenses/MIT)

Copyright (c) 2016-2017 Dave Henderson

[![Analytics](https://ga-beacon.appspot.com/UA-82637990-1/gomplate/docs/README.md?pixel)](https://github.com/igrigorik/ga-beacon)