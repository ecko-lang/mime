# mime - Ecko Std Lib Package

Build MIME messages and look up media types, in pure Ecko.

Two jobs that keep turning up together: assembling an email with a text body, an
HTML alternative and attachments, and answering "what media type is this
filename" for uploads and static serving.

Pure computation - no capabilities. Reading a file to attach is the caller's
job, and their `fs:read` grant, not this package's.

## Install

```bash
ecko get github.com/ecko-lang/mime
```

`ecko get` vendors the package under `./vendor/github.com/ecko-lang/mime/` and
pins its checksum in `ecko.lock`.

```ecko
import mime
```

## Usage

Parts are added with `|>`, and nothing is mutated in place:

```ecko
import mime
import smtp

msg = mime.message({ from: "reports@example.com", to: "ada@example.com", subject: "Daily numbers" })
    |> mime.text("Everything is green. The detail is attached.")
    |> mime.html("<p>Everything is <strong>green</strong>.</p>")
    |> mime.attach("report.pdf", pdf_bytes)

smtp.send(server, mime.render(msg))
```

`render` returns exactly the map `smtp.send` expects, so the two compose without
either package knowing about the other.

Media types, for uploads and static serving:

```ecko
mime.type_for("photo.png")        # "image/png"
mime.type_for("/srv/app.js")      # "text/javascript"
mime.type_for("notes")            # "application/octet-stream"
mime.extension_for("image/jpeg")  # "jpg"
```

## API

### Building

| function | what it does |
|---|---|
| `message(spec)` | start a message from `{ from, to, subject, date?, headers? }`; `to` may be a string or a list |
| `text(msg, body)` | add a `text/plain` part |
| `html(msg, body)` | add a `text/html` part |
| `attach(msg, filename, data)` | add an attachment, media type inferred from the extension |
| `attach_as(msg, filename, data, content_type)` | as `attach`, with the type given explicitly |
| `add_part(msg, part)` | append a prepared part, for shapes this package does not model |

### Rendering

| function | what it does |
|---|---|
| `render(msg)` | `{ from, to, subject, headers, body }` - the map `smtp.send` takes |
| `to_string(msg)` | the whole message as RFC 5322 text, for a `.eml` or a fixture |
| `boundary_of(msg)` | the multipart delimiter this message will use |
| `subtype_of(msg)` | `"mixed"`, `"alternative"`, or `null` when it is a single part |

### Media types

| function | what it does |
|---|---|
| `type_for(path)` | media type from a filename or path, `application/octet-stream` if unknown |
| `extension_for(content_type)` | the preferred extension, or `null` |
| `table()` | the extension-to-type table |
| `preferred_extensions()` | the type-to-extension table used by `extension_for` |
| `trim_type(ct)` | `"text/plain; charset=utf-8"` to `"text/plain"` |

## Notes

**The structure follows the content.** One text part renders as a plain
single-part message. Text plus HTML becomes `multipart/alternative`. Anything
with an attachment becomes `multipart/mixed`. You never ask for a multipart; you
get one when the parts need it.

**Boundaries are derived, not random.** The delimiter is a hash of the parts, so
it cannot collide with content that contains it, and the same message renders
byte-identically every time. That is what makes rendered messages diffable and
the tests able to assert on exact output.

**Header injection is handled.** CR and LF are stripped from every header value,
so a newline smuggled into a subject cannot start an attacker-chosen header - the
email equivalent of HTTP response splitting. Attachment filenames additionally
have `\` and `"` escaped so they cannot end the quoted parameter early.

**Attachments are base64, wrapped at 76 columns.** RFC 2045 caps an encoded line
at 76 characters and RFC 5322 caps any line at 998 octets, so an unwrapped
attachment is not merely untidy: some servers refuse it.

**Sending needs smtp 0.9.6 or later.** Earlier versions always appended their own
`Content-Type: text/plain`, which collides with the multipart type this package
sets. `to_string` is unaffected.

## Testing

```bash
ecko test
```

Offline and deterministic - no network, no clock, no randomness. `example.ecko`
runs the whole flow including a mock SMTP transport, and prints the same bytes
every time.

## License

MIT
