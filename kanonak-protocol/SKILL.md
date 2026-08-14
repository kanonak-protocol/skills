---
name: kanonak-protocol
description: Read, write, and validate Kanonak Protocol ontologies (.kan.yml files). Covers the document format, the publisher/package@version/name URI scheme and how it maps to fetchable URLs, installing and driving the kanonak CLI, and where the canonical modeling and styling guides live. Use when working with .kan.yml files, authoring or reviewing a Kanonak ontology, resolving Kanonak package imports or versions, running the kanonak CLI, publishing a Kanonak package, or styling one with the look system.
license: Apache-2.0
compatibility: Reading published packages needs only HTTPS access to publisher domains. Authoring and validating additionally need Node.js and npm to install the kanonak CLI.
---

# Kanonak Protocol

Kanonak is an open protocol for defining, versioning, and sharing semantic
ontologies across independent publishers. There is no central registry — each
publisher serves its own packages from its own domain over plain HTTPS.

Start here, then follow the pointers in "Learn the rules" below. The protocol
documents itself: the conventions you need are published as Kanonak packages
you can fetch and read, so they stay current as the protocol evolves.

## The one idea to understand first

Every class, property, and instance has a stable URI:

```
publisher/package@version/name
kanonak.org/core-rdf@1.1.0/Class
```

That URI maps directly onto a URL you can fetch:

```
https://kanonak.org/core-rdf/1.1.0.kan.yml
```

A package is a **single YAML file** named `<version>.kan.yml`, living in a
directory named after the package. Resolution is just HTTP — nothing else.
Anyone who can serve a file can publish an ontology.

## A complete document

This is a whole, valid package. Nothing is elided.

```yaml
bookshelf:
  type: Package
  publisher: example.com
  version: 1.0.0
  label: Bookshelf
  comment: A tiny example ontology describing books and their authors.
  imports:
    - publisher: kanonak.org
      packages:
        - package: core-rdf
          match: ^
          version: 1.0.0
          alias: rdfs
        - package: core-owl
          match: ^
          version: 2.0.0
          alias: owl
        - package: core-xsd
          match: ^
          version: 1.0.0
          alias: xsd

Author:
  type: rdfs.Class
  label: Author
  comment: A person who writes books.

Book:
  type: rdfs.Class
  label: Book
  comment: A published book.

authorName:
  type: owl.DatatypeProperty
  domain: Author
  range: xsd.string
  label: Name
  comment: The author's full name.

title:
  type: owl.DatatypeProperty
  domain: Book
  range: xsd.string
  label: Title
  comment: The book's title.

writtenBy:
  type: owl.ObjectProperty
  domain: Book
  range: Author
  label: Written By
  comment: The author who wrote this book.

ursula-le-guin:
  type: Author
  authorName: Ursula K. Le Guin

a-wizard-of-earthsea:
  type: Book
  title: A Wizard of Earthsea
  writtenBy: ursula-le-guin
```

Saved as `bookshelf/1.0.0.kan.yml`, that validates with zero errors.

Reading it:

- **The first key is the package header.** Its name matches the directory. It
  declares `publisher`, `version`, and `imports`.
- **Every other top-level key is a resource** in the package. What it *is* comes
  from its `type`.
- **Imports get a document-local `alias`.** `rdfs` here is just this file's
  nickname for `kanonak.org/core-rdf`. Another document may call the same
  package something else. Aliases are never global — always resolve them
  through the file's own `imports` block.
- **`match: ^`** is the semver operator: accept any compatible version at or
  above `1.0.0`. `~` is minor-compatible, `=` is exact, `*` is any.
- **References are bare names.** `writtenBy`'s `range: Author` points at the
  `Author` in this document; `range: xsd.string` crosses into an imported
  package via its alias.
- **Classes and instances live side by side.** `Book` is a class;
  `a-wizard-of-earthsea` is an instance of it. Same file, same syntax.

## Get the CLI

```bash
npm install -g @kanonak-protocol/cli
kanonak --help
```

`kanonak --help` is authoritative for the command surface. If anything below
disagrees with it, believe `--help`.

## The authoring loop

Write, validate, read the error, fix, repeat:

```bash
kanonak validate bookshelf/1.0.0.kan.yml
kanonak validate .
```

Validation resolves imports over HTTP against the real publisher domains, so
it checks your references genuinely exist — not just that your YAML parses.

To see what a document actually resolved to:

```bash
kanonak deps bookshelf/1.0.0.kan.yml
```

```
example.com/bookshelf@1.0.0
  kanonak.org/core-rdf@1.1.0
  kanonak.org/core-owl@2.2.0
    kanonak.org/core-xsd@1.1.0
  kanonak.org/core-xsd@1.1.0
```

Note the resolved versions differ from the `^` floors written in the file.
That is the operator doing its job.

## Learn the rules

The protocol's conventions are themselves published packages. Install one and
read it:

```bash
kanonak install kanonak.org/ontology-conventions
```

It lands in `~/.kanonak/packages/kanonak.org/`. Without the CLI, fetch the
same content directly:

```bash
curl https://kanonak.org/ontology-conventions/1.1.0.kan.yml
```

Or read the rendered page: <https://kanonak.org/ontology-conventions/ontology-conventions-spec>

Each guide is a `Protocol` resource with a `hasConvention` block. Every
convention carries required and recommended rules, each with a `rationale`,
plus worked valid and invalid examples. When the validator cites a rule by
name, find that rule in the installed file and read it directly.

**`kanonak.org/ontology-conventions`** — how to *model*. Read this before
designing anything. Covers: classes vs. properties vs. individuals; datatype
vs. object properties; `subClassOf` and multiple inheritance; when an
individual belongs to the schema as an enumeration versus being data;
constraining properties with shapes.

**`kanonak.org/kanonak-protocol`** — the foundational spec every document
obeys regardless of vocabulary. Covers URI structure and canonical URL form,
fragments and query strings, publisher/package/resource naming, versioning and
file naming, import operators and version resolution, the type system,
embedding, references, hierarchy, and the canonical structural hash.

Available versions for any package are listed at
`https://kanonak.org/<package>/`.

## Styling with look

Kanonak packages render as web pages. The **look** system is the declarative
layer that controls how — you style the ontology by adding look declarations
to the graph, not by writing a renderer.

Start with the conventions guide, which is also the decision guide for when a
look is the right tool versus a transformation or a view:

```bash
kanonak install kanonak.org/look-conventions
```

<https://kanonak.org/look-conventions/look-conventions-spec>

It covers: choosing an approach; styling types rather than instances; the
resource view; path carrier bands; visual identity; display lenses; and the
cascade and its universal floor.

The vocabulary itself is `kanonak.org/look`. To render locally:

```bash
kanonak derive <resource-uri>
kanonak serve
```

## Things that trip people up

- **Aliases are document-local.** Never assume `rdfs` means the same package in
  two files, and never derive meaning from a name's prefix. Resolve through the
  file's `imports`.
- **Identity is the full URI**, not the bare name. Two packages can each define
  `Author`; they are different things.
- **Published versions are immutable.** Fix a mistake by publishing a new
  version, never by editing one that is already out.
- **An unresolved reference is an error, not an absence.** If something does not
  resolve, that is a bug to surface — do not work around it by guessing or by
  falling back to a string.
- **Check the `.kan.yml` before assuming a package's shape.** Fetching it is one
  HTTP request and settles the question.
