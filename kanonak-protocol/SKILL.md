---
name: kanonak-protocol
description: Read, write, and validate Kanonak Protocol ontologies (.kan.yml files). Covers the document format, separating schema (classes and properties) from data (instances) into their own packages, the publisher/package@version/name URI scheme and how it maps to fetchable URLs, installing and driving the kanonak CLI, and where the canonical modeling and styling guides live. Use when working with .kan.yml files, authoring or reviewing a Kanonak ontology or its instance data, resolving Kanonak package imports or versions, running the kanonak CLI, publishing a Kanonak package, or styling one with the look system.
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

Two different URLs come off that URI, and it matters that you keep them apart.

### The canonical resource URL

Pure structural substitution — swap `@` for `/`, prefix `https://`:

```
https://kanonak.org/core-rdf/1.1.0/Class
```

This mapping is bijective and never requires asking the publisher anything, in
either direction. It is also the page a browser renders. Five forms address
every target, and publishers may not invent others:

```
/                              publisher
/{package}                     package, any version
/{package}/{version}           package, pinned
/{package}/{name}              resource, any version
/{package}/{version}/{name}    resource, pinned
```

### The package source URL

Where the raw YAML *bytes* live. That is a separate question, and the answer is
publisher-configurable. The default:

```
https://{publisher}/{package}/{version}.kan.yml
https://kanonak.org/core-rdf/1.1.0.kan.yml
```

But a publisher can advertise a different layout in `.well-known/kanonak.json`:

```json
{
  "version": 1,
  "package_url_template": "https://cdn.example.com/{publisher}/{package}-{version}.yaml"
}
```

When `package_url_template` is present it wins. The contract is that filling
`{publisher}`, `{package}`, and `{version}` yields an `https://` URL whose body
is the package source. Beyond that, host, path shape, separators, and file
extension are all free — which is what lets a publisher delegate hosting to a
CDN. When the config is absent, or omits the field, the default template
applies.

A template MUST use all three placeholders — `{publisher}`, `{package}`, and
`{version}`. Do not hardcode any of them, even when a value looks fixed for
your own site. A template that carries all three resolves for any publisher,
which is what lets the same shape serve a shared archive, a multi-tenant host,
or the protocol's own default:

```
https://{publisher}/{package}/{version}.kan.yml
```

**So: derive canonical resource URLs freely, but never hardcode a source URL for
a publisher you do not control.** Let the CLI resolve it — it reads the
publisher's config for you. A package is one YAML file per version, and
resolution is just HTTP, so anyone who can serve a file can publish an ontology.

## Separate the schema from the data

Before writing anything, decide which of two things you are authoring:

- **Schema** — the model. What a `Book` *is*: classes and the properties they
  carry. Changes rarely.
- **Data** — instances. The actual books on the shelf. Changes constantly.

These belong in **separate packages**, because they have different lifecycles.
Bundled together, every routine data update forces a new version of the schema,
which is noisy and breaks the pinning your consumers rely on. Split, each
versions at its own pace and many data packages can share one schema.

(If you know the description-logic terms, this is TBox vs. ABox. The protocol
just calls it schema and data.)

### The schema package

`bookshelf/1.0.0.kan.yml` — classes and properties, no content.

```yaml
bookshelf:
  type: Package
  publisher: example.com
  version: 1.0.0
  label: Bookshelf
  comment: The schema — what a book and an author are.
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

Format:
  type: rdfs.Class
  owl.oneOf: [ hardcover, paperback, ebook ]
  label: Format
  comment: The fixed set of formats a book is published in.

hardcover:
  type: Format
  label: Hardcover

paperback:
  type: Format
  label: Paperback

ebook:
  type: Format
  label: E-book

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

format:
  type: owl.ObjectProperty
  domain: Book
  range: Format
  label: Format
  comment: The format this book was published in.
```

### The data package

`bookshelf-library/1.0.0.kan.yml` — imports the schema, asserts instances of
its classes. Nothing else.

```yaml
bookshelf-library:
  type: Package
  publisher: example.com
  version: 1.0.0
  label: Bookshelf Library
  comment: The data — actual books and authors, asserted against the schema.
  imports:
    - publisher: example.com
      packages:
        - package: bookshelf
          match: ^
          version: 1.0.0
          alias: bs

ursula-le-guin:
  type: bs.Author
  bs.authorName: Ursula K. Le Guin

a-wizard-of-earthsea:
  type: bs.Book
  bs.title: A Wizard of Earthsea
  bs.writtenBy: ursula-le-guin
  bs.format: bs.ebook
```

Both files together validate with zero errors. The parser merges them into one
logical graph at load time, so a consumer that loads both sees `Book` and its
instances as a single connected model.

### Reading these

- **The first key is the package header.** Its name matches the directory. It
  declares `publisher`, `version`, and `imports`.
- **Every other top-level key is a resource.** What it *is* comes from its
  `type`.
- **Imports get a document-local `alias`.** `rdfs` is just this file's nickname
  for `kanonak.org/core-rdf`; `bs` is the data package's nickname for the
  schema. Another document may pick different names for the same packages.
  Aliases are never global — always resolve them through the file's own
  `imports` block.
- **`match: ^`** is the semver operator: accept any compatible version at or
  above `1.0.0`. `~` is minor-compatible, `=` is exact, `*` is any.
- **Unprefixed names are local; prefixed names cross a package boundary.** In
  the schema, `range: Author` means the `Author` in that same file. In the data
  package, `type: bs.Book` reaches into the imported schema.

### The one exception: enumerations

`Format` above is a class *with* named individuals, sitting in the schema — and
that is correct. A closed, fixed set of members (formats, statuses, categories,
units) is part of the model, not data.

The protocol never guesses which is which. An individual is schema **only** when
explicitly marked: `owl.oneOf` on the class for a closed set of named members,
or `sh.in` on a property for a closed set of literal values. A class that merely
happens to have instances is not an enumeration — its instances are data.

So: `hardcover` is schema because `Format` declares `owl.oneOf`.
`a-wizard-of-earthsea` is data because `Book` does not.

Small vocabularies may legitimately mix a few canonical instances into the
schema package. At any real scale, keep them apart.

## What building one looks like

A worked session. The shape to aim for is: establish purpose, agree the schema,
validate, then add data.

---

**AI** — What kind of ontology would you like to build?

**User** — An ontology about cars.

**AI** — Two questions first, because the answers change the shape of the model:

1. What will you do with it — catalogue production models, track individual
   vehicles, compare specifications?
2. Is there a fixed set of anything (fuel types, body styles), or is everything
   open-ended?

**User** — Catalogue production models. Fuel type is a fixed list: petrol,
diesel, electric.

**AI** — Then `FuelType` is an enumeration and belongs in the schema, while the
cars themselves are data. Starting with the schema package — classes and
properties only, no actual cars in it:

```yaml
cars:
  type: Package
  publisher: example.com
  version: 1.0.0
  label: Cars
  comment: The schema — what a car and a manufacturer are.
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

Manufacturer:
  type: rdfs.Class
  label: Manufacturer
  comment: A company that builds cars.

Car:
  type: rdfs.Class
  label: Car
  comment: A production car model.

FuelType:
  type: rdfs.Class
  owl.oneOf: [ petrol, diesel, electric ]
  label: Fuel Type
  comment: The fixed set of fuel types a car runs on.

petrol:
  type: FuelType
  label: Petrol

diesel:
  type: FuelType
  label: Diesel

electric:
  type: FuelType
  label: Electric

manufacturerName:
  type: owl.DatatypeProperty
  domain: Manufacturer
  range: xsd.string
  label: Name

modelName:
  type: owl.DatatypeProperty
  domain: Car
  range: xsd.string
  label: Model Name

builtBy:
  type: owl.ObjectProperty
  domain: Car
  range: Manufacturer
  label: Built By

fuel:
  type: owl.ObjectProperty
  domain: Car
  range: FuelType
  label: Fuel
```

```
$ kanonak validate cars/1.0.0.kan.yml
1 file(s) validated. 0 error(s), 0 warning(s), 0 info(s).
```

`FuelType` carries `owl.oneOf`, so its three members are part of the model.
`Car` carries no such marker, so actual cars are data and belong elsewhere.

**User** — Add the Golf GTI and the ID. Buzz, both Volkswagen.

**AI** — Those are data, so they go in their own package that imports the
schema:

```yaml
cars-inventory:
  type: Package
  publisher: example.com
  version: 1.0.0
  label: Cars Inventory
  comment: The data — actual car models.
  imports:
    - publisher: example.com
      packages:
        - package: cars
          match: ^
          version: 1.0.0
          alias: c

volkswagen:
  type: c.Manufacturer
  c.manufacturerName: Volkswagen

golf-gti:
  type: c.Car
  c.modelName: Golf GTI
  c.builtBy: volkswagen
  c.fuel: c.petrol

id-buzz:
  type: c.Car
  c.modelName: ID. Buzz
  c.builtBy: volkswagen
  c.fuel: c.electric
```

```
$ kanonak validate .
2 file(s) validated. 0 error(s), 0 warning(s), 0 info(s).
```

The inventory can now grow — new models, new manufacturers — without the schema
leaving 1.0.0.

---

Follow that order: **purpose → schema → validate → data → validate.** Ask what
the ontology is *for* before choosing classes. The answer is what decides which
things are enumerations and which are data, and that decision is far cheaper to
make now than to unpick after the schema is published.

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

Validate the whole workspace with `.` when packages reference each other — that
resolves siblings locally, so the data package finds its schema before either
one is published.

Validation is not a YAML syntax check. It resolves every import and every
reference, fetching published packages over HTTP from their publisher domains.
A name that does not resolve is an error, with the fix spelled out:

```
example.com/bookshelf-library@1.0.0:
  ERROR: Reference to 'audiobook' in 'format' could not be resolved
    -> The entity 'example.com/bookshelf/audiobook' is not defined in this
       namespace or any imported namespace.
  • Import the package that defines it, or check for a typo in 'audiobook'
  • If it lives in an imported namespace, alias-qualify the reference

2 file(s) validated. 1 error(s), 0 warning(s), 0 info(s).
```

Read those errors literally — they name the exact URI that failed to resolve.
Do not work around one by guessing a name or leaving a value as a plain string.

To see what a published document actually resolved to:

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

It lands in `~/.kanonak/packages/kanonak.org/`. Without the CLI, read the
rendered page — a canonical resource URL, so it is always derivable:

<https://kanonak.org/ontology-conventions/ontology-conventions-spec>

Or fetch the source directly. kanonak.org serves at the default template, so
this works — but confirm `.well-known/kanonak.json` first for any publisher
whose layout you have not checked:

```bash
curl https://kanonak.org/ontology-conventions/1.1.0.kan.yml
```

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

Available versions for any package are listed at its canonical package URL —
`https://kanonak.org/ontology-conventions` for this one.

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

- **Do not add instances to a schema package** because it is convenient. That
  couples the model's version to the data's churn, which is the coupling the
  split exists to prevent.
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
- **Do not assume where a publisher's source files live.** `<package>/<version>.kan.yml`
  is the default, not a guarantee — `.well-known/kanonak.json` can point source
  bytes anywhere. Canonical *resource* URLs are always structural; *source* URLs
  are not.
- **Check the `.kan.yml` before assuming a package's shape.** Fetching it is one
  HTTP request and settles the question.
