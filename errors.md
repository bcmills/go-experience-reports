# Error Wrapping and Redundancy in Go

A Go [experience report](http://golang.org/wiki/ExperienceReports).

## The error-wrapping idiom.

The convention in Go, as illustrated in the
[Error handling and Go blog post](https://blog.golang.org/error-handling-and-go)
and [Effective Go](https://golang.org/doc/effective_go.html#errors), is that
“[i]t is the error implementation's responsibility to summarize the context.”

The blog post gives an example:

```go
func Sqrt(f float64) (float64, error) {
    if f < 0 {
        return 0, fmt.Errorf("math: square root of negative number %g", f)
    }
    […]
}
```

Effective Go uses a similar example, wherein `os.Create(filename)` returns an
`*os.PathError` that includes the name of the function (“create”, in the `Op`
field), the contents of the `filename` argument (in the `Path` field), and the
underlying error detail (in the `Err` field).

The general rule can be summarized as follows: When returning an error from a
call, describe all relevant arguments.

## A case study: the `go` command.

Adding detail on the callee side works out pretty nicely for simple call sites:
because the callee has already added the details, in many cases one can simply
`return err` at each error without any additional wrapping. But what happens
when you need to wrap errors from calls needing to add detail to errors at
multiple levels. Then that strategy turns into a flood of redundant information.
To take an example from the `go` command itself, taken during the Go 1.13
development cycle:

```
example.com$ GO111MODULE=on gotip build golang.org/x/tools/does/not/exist
go: finding golang.org/x/tools latest
can't load package: package golang.org/x/tools/does/not/exist: unknown import path "golang.org/x/tools/does/not/exist": module golang.org/x/tools@latest (v0.0.0-20190614152001-1edc8e83c897) found, but does not contain package golang.org/x/tools/does/not/exist
```

That last line is a wrapped error. It repeats the word “package” and the import
path that produced the error three times each, and the reader ends up lost in a
jumble of text. A better error might look something like:

```
package golang.org/x/tools/does/not/exist: module golang.org/x/tools@latest found (v0.0.0-20190614152001-1edc8e83c897), but does not contain package
```

How did we end up with the redundant error message, and how could we get to the
better one?

We ended up with the redundant message by the following (condensed) call chain:
`load.Packages` calls `load.loadImport`, and writes the resulting error to
`stderr` with additional detail. `load.loadImport` calls `load.loadPackageData`,
and adds detail by wrapping the error in a `*load.PackageError`
`load.loadImport` calls `modload.Lookup`, and adds detail using `fmt.Errorf`.
`modload.Lookup` returns an error previously cached by
`(*modload.loader).doPkg`. `(*modload.loader).doPkg` calls `modload.Import`.
`modload.Import` calls `modload.QueryPackage`. `modload.QueryPackage` captures
detail in a `*modload.packageNotInModuleError`.

With the call chain in mind, we can now understand the structure of the error
message. The `can't load package:` prefix comes from `load.Packages`. The
`package golang.org/x/tools/does/not/exist:` prefix comes from the `Error`
method of `*load.PackageError`, which follows the example of `*os.PathError`,
with one subtle difference: `*load.PackageError` adds the word `package` itself,
whereas `*os.PathError` does not add the word `path`. The `unknown import path
"golang.org/x/tools/does/not/exist":` prefix comes from `load.loadPackageData`.
The final mention of `package golang.org/x/tools/does/not/exist` comes from the
`Error` method of `*modload.packageNotInModuleError`.

The problem here is that redundant information has been added in multiple
layers. To avoid that problem while still following the usual Go idiom, we must
add an exception to our rule: When returning an error from a call, describe all
relevant arguments…. _...except_ those that were passed to a call from which the
underlying error originated.

So let's try that approach. `modload.Lookup`, `modload.QueryPackage` and
`modload.Import` all receive the path and know that it is a package path, so
`load.loadPackageData` must not add the path, nor the information that it is a
package path, to the result of the `modload.Lookup` call. So let's take out that
`fmt.Errorf`. `load.loadImport` passed all of that information to
`load.loadPackageData`, so `load.loadImport` must not include that information
in its error. However, `loadImport` has additional information that may be
relevant — namely, the import stack leading to the package, or lack thereof. The
only new information that `load.Packages` has is the mapping from arguments
(package patterns) to specific packages, which it should only include if not
redundant with the package path itself.

If we apply those changes, we end up with an error message that looks something
like:

```
module golang.org/x/tools@latest found (v0.0.0-20190614152001-1edc8e83c897), but does not contain package golang.org/x/tools/does/not/exist
```

That's not too bad, but in our quest to prune out _redundant_ information, we
have lost control over _where_ in our error message that information appears: we
would like to lead with the package path, but this error message begins with the
module information instead. The nesting structure of the call chain directly
dictates the structure of the error message. If we want to change that nesting
structure, we need to _unwrap and re-wrap_ the errors at some point in the
chain, or else we need to refactor the most deeply-nested call to change the
emphasis of its error message globally.

In other situations, we may encounter similar problems of information control:
information relevant to one caller may not be relevant to all callers, but the
callee does not have enough information to distinguish them. For example, in the
context of an HTTP server, the file path involved in a filesystem operation may
be completely redundant with the URL passed to the HTTP handler. The URL is
probably the more relevant information for the maintainer of the server, as that
includes additional information such as query parameters. When logging an error
from such a handler, the author must either include the redundant path, or
attempt to filter it from a (possibly very deeply nested) `*os.PathError` at the
bottom of the error chain.

### Addendum: closure caveats.

With this idiom, the caller might not actually know what information is
available to the callee, and thus might not know which information is redundant.
For example, if an argument to a function is itself callable (such as an
interface value or a function variable), the callee may or may not know about
arguments that are already contained in the receiver or function closure.

Fortunately, in practice the number of cases where this ambiguity occurs tend to
be small, and when they do occur the number of possible wrapper types also tends
to be small: with care, we can unpack the error, detect (or remove) the
redundant information, and then re-pack it with minimal redundancy.
