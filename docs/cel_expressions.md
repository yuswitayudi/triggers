<!--
---
linkTitle: "CEL Expression Extensions"
weight: 8
---
-->
# CEL expression extensions

The CEL expression is configured to expose parts of the request, and some custom
functions to make matching easier.

In addition to the custom function extension listed below, you can craft any
valid CEL expression as defined by the
[cel-spec language definition](https://github.com/google/cel-spec/blob/master/doc/langdef.md)

## String functions

The [upstream CEL implementation](https://github.com/google/cel-go/) provides
extensions to the CEL specification for manipulating strings.

For example:

```javascript
'refs/heads/master'.split('/') // result = list ['refs', 'heads', 'master']
'my place'.replace('my ',' ') // result = string 'place'
'this that another'.replace('th ',' ', 2) // result = 'is at another'
```

The `replace` overload allows an optional limit on replacements.

## Notes on numbers in CEL expressions

One thing to be aware of is how numeric values are treated in CEL expressions,
JSON numbers are decoded to
[CEL double](https://github.com/google/cel-spec/blob/master/doc/langdef.md#values)
values.

For example:

```json
{
  "count": 2,
  "measure": 1.7
}
```

In the JSON above, both numbers are parsed as CEL double (Go float64) values.

This means that if you want to do integer arithmetic, you'll need to
[use explicit conversion functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#numeric-values).

From the CEL specification:

> Note that currently there are no automatic arithmetic conversions for the
> numeric types (int, uint, and double).

You can either explicitly convert the number, or add another double value e.g.

```yaml
interceptors:
  - cel:
      overlays:
        - key: count_plus_1
          expression: "body.count + 1.0"
        - key: count_plus_2
          expression: "int(body.count) + 2"
        - key: measure_times_3
          expression: "body.measure * 3.0"
```

These will be serialised back to JSON appropriately:

```json
{
  "count_plus_1": 2,
  "count_plus_2": 3,
  "measure_times_3": 5.1
}
```

### Error messages in conversions

The following example will generate an error with the JSON example.

```yaml
interceptors:
  - cel:
      overlays:
        - key: bad_measure_times_3
          expression: "body.measure * 3"
```

**bad_measure_times_3** will fail with
`failed to evaluate overlay expression 'body.measure * 3': no such overload`
because there's no automatic conversion.

## CEL expression examples

### Matching on an element in an array.

CEL provides several [macros](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros) which can operate on JSON objects.

If you have a JSON body like this:

```json
{
  "labels": [
    {
      "name": "test-a"
    },
    {
      "name":"test-b"
    }
  ]
}
```

You can use this in filters in the following ways:

 * `filter: body.labels.exists(x, x.name == 'test-b')` is _true_
 * `filter: body.labels.exists(x, x.name == 'test-c')` is _false_
 * `filter: body.labels.exists_one(x, x.name.endsWith('-b'))` is _true_
 * `filter: body.labels.exists_one(x, x.name.startsWith('test-'))` is _false_
 * `filter: body.labels.all(x, x.name.startsWith('test-'))` is _true_
 * `filter: body.labels.all(x, x.name.endsWith('-b'))` is _false_

You can also parse additional data from each of the labels:
```yaml
overlays:
- key: suffixes
  expression: "body.labels.map(x, x.name.substring(x.name.lastIndexOf('-')+1))"
```
This yields an array of `["a", "b"]` in the `suffixes` extension key.
```yaml
overlays:
- key: filtered
  expression: "body.labels.filter(x, x.name.endsWith('-b'))"
```
This would add an extensions key `filtered` with only one of the labels.
```yaml
[
  {
    "name": "test-b"
  }
]
```

## cel-go extensions

All the functionality from the cel-go project's [CEL extension](https://github.com/google/cel-go/tree/master/ext) is available in
your CEL expressions.

### cel-go Bytes

The cel-go project function `base64.decode` returns a [CEL `Bytes`](https://github.com/google/cel-spec/blob/master/doc/langdef.md#string-and-bytes-values) value.

To compare this to a string, you will need to convert it to a Bytes type:

```
base64.decode(body.b64value) == b'hello' # compare to Bytes literal
base64.decode(body.b64value) == bytes('hello') # convert to bytes.
```

### Returning Bytes

Confusingly, if you decode a base64 string with the cel-go base64 decoder, it will
appear in the extension as a base64 encoded string, you will need to explicitly
convert it to a CEL string.

```yaml
interceptors:
  - cel:
      overlays:
        - key: base64_decoded
          expression: "string(base64.decode(body.b64Value))"
```
This will correctly appear in the extension as the decoded version.

## List of extensions

The body from the `http.Request` value is decoded to JSON and exposed, and the
headers are also available.

<table style="width=100%" border="1">
  <tr>
    <th>Symbol</th>
    <th>Type</th>
    <th>Description</th>
    <th>Example</th>
  </tr>
  <tr>
    <th>
      body
    </th>
    <td>
      map(string, dynamic)
    </td>
    <td>
      This is the decoded JSON body from the incoming http.Request exposed as a map of string keys to any value types.
    </td>
    <td>
      <pre>body.value == 'test'</pre>
    </td>
  </tr>
  <tr>
    <th>
      header
    </th>
    <td>
      map(string, list(string))
    </td>
    <td>
      This is the request Header.
    </td>
    <td>
      <pre>header['X-Test'][0] == 'test-value'</pre>
    </td>
  </tr>
  <tr>
    <th>
      requestURL
    </th>
    <td>
      string
    </td>
    <td>
      This is the URL for the incoming HTTP request.
    </td>
    <td>
      <pre>requestURL.parseURL().path</pre>
    </td>
  </tr>
</table>

NOTE: The header value is a Go `http.Header`, which is
[defined](https://golang.org/pkg/net/http/#Header) as:

```go
type Header map[string][]string
```

i.e. the header is a mapping of strings, to arrays of strings, see the `match`
function on headers below for an extension that makes looking up headers easier.

## List of extension functions

This lists custom functions that can be used from CEL expressions in the CEL
interceptor.

<table style="width=100%" border="1">
  <tr>
    <th>Symbol</th>
    <th>Type</th>
    <th>Description</th>
    <th>Example</th>
  </tr>
  <tr>
    <th>
      match
    </th>
    <td>
      header.match(string, string) -> bool
    </td>
    <td>
      Uses the canonical header matching from Go's http.Request to match the header against the value.
    </td>
    <td>
     <pre>header.match('x-test', 'test-value')</pre>
    </td>
  </tr>
  </tr>
    <th>
      canonical
    </th>
    <td>
      header.canonical(string) -> string
    </td>
    <td>
      Uses the canonical header matching from Go's http.Request to get the provided header name.
    </td>
    <td>
     <pre>header.canonical('x-test')</pre>
    </td>
  </tr>
  <tr>
    <th>
      truncate
    </th>
    <td>
      <pre>&lt;string&gt;.truncate(uint) -> string</pre>
    </td>
    <td>
      Truncates a string to no more than the specified length.
    </td>
    <td>
     <pre>body.commit.sha.truncate(5)</pre>
    </td>
  </tr>
  <tr>
    <th>
      split
    </th>
    <td>
      <pre>&lt;string&gt;.split(string) -> string(dyn)</pre>
    </td>
    <td>
      Splits a string on the provided separator value.
    </td>
    <td>
     <pre>body.ref.split('/')</pre>
    </td>
  </tr>
  <tr>
    <th>
      decodeb64 **deprecated: please use base64.decode**
    </th>
    <td>
      <pre>&lt;string&gt;.decodeb64() -> string</pre>
    </td>
    <td>
      Decodes a base64 encoded string.
    </td>
    <td>
     <pre>body.message.data.decodeb64()</pre>
    </td>
  </tr>
  <tr>
    <th>
     compareSecret
    </th>
    <td>
      <pre>&lt;string&gt;.compareSecret(string, string, string) -> bool</pre>
    </td>
    <td>
      Constant-time comparison of strings against secrets, this will fetch the secret using the combination of namespace/name and compare the token key to the string using a cryptographic constant-time comparison..<p>
      The event-listener service account must have access to the secret.
      The parameters to the function are 1. the key within the secret, 2. the secret name, and 3. the namespace for the secret (optional, defaults to the namespace of the EventListener).
    </td>
    <td>
     <pre>header.canonical('X-Secret-Token').compareSecret('', 'secret-name', 'namespace')</pre>
    </td>
  </tr>
  <tr>
    <th>
     compareSecret
    </th>
    <td>
      <pre>&lt;string&gt;.compareSecret(string, string) -> bool</pre>
    </td>
    <td>
     This is almost identical to the version above, but only requires two arguments, the namespace is assumed to be the namespace for the event-listener.
    </td>
    <td>
     <pre>header.canonical('X-Secret-Token').compareSecret('key', 'secret-name')</pre>
    </td>
  </tr>
  <tr>
    <th>
     parseJSON()
    </th>
    <td>
     <pre>&lt;string&gt;.parseJSON() -> map&lt;string, dyn&gt;</pre>
    </td>
    <td>
     This parses a string that contains a JSON body into a map which which can be subsequently used in other expressions.
    </td>
    <td>
     <pre>'{"testing":"value"}'.parseJSON().testing == "value"</pre>
    </td>
  </tr>
  <tr>
    <th>
     parseYAML()
    </th>
    <td>
     <pre>&lt;string&gt;.parseYAML() -> map&lt;string, dyn&gt;</pre>
    </td>
    <td>
     This parses a string that contains a YAML body into a map which which can be subsequently used in other expressions.
    </td>
    <td>
     <pre>'key1: value1\nkey2: value2\n'.parseYAML().key1 == "value"</pre>
    </td>
  </tr>
  <tr>
    <th>
     parseURL()
    </th>
    <td>
     <pre>&lt;string&gt;.parseURL() -> map&lt;string, dyn&gt;</pre>
    </td>
    <td>
     This parses a string that contains a URL into a map with keys for the elements of the URL.<br />
     The resulting map will contain the following keys for this URL "https://user:pass@example.com/test/path?s=testing#first"<br />
     <table>
      <tr><th>Field</th><th>Example</th>
      <tr><td>scheme</td><td>https</td></tr>
      <tr><td>host</td><td>example.com</td></tr>
      <tr><td>path</td><td>/test/path</td></tr>
      <tr><td>rawQuery</td><td>s=testing</td></tr>
      <tr><td>fragment</td><td>first</td></tr>
      <tr><td>query</td><td>{"s": "testing"}</td></tr>
      <tr><td>queryStrings</td><td>{"s": ["testing"]}</td></tr>
      <tr><td>auth</td><td>{"username": "user", "password": "pass"}</td></tr>
     </table>
     Note the difference between <b>query</b> and <b>queryStrings</b>, in
<b>query</b>, multiple query params with the same name would be comma separated, for
the case where a single string is provided, this will just be the single string
value.  For <b>queryString</b> the query param values are provided as a list,
which can be accessed by indexing.
    </td>
    <td>
     <pre>'https://example.com/test?query=testing'.parseURL().query['query'] == "testing"</pre>
    </td>
  </tr>
  <tr>
    <th>
     marshalJSON()
    </th>
    <td>
     <pre>&lt;jsonObjectOrList&gt;.marshalJSON() -> &lt;string&gt;</pre>
    </td>
    <td>
     Returns the JSON encoding of 'jsonObjectOrList' as a string.
    </td>
    <td>
     <pre>{"testing":"value"}.marshalJSON() == "{\"testing\": \"value\"}"</pre>
    </td>
  </tr>
</table>
