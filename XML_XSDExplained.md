# XML & XSD — Foundations Reference

A from-scratch reference for understanding XML and XSD, built while reading the
ISO 20022 `pain.001.001.13` schema. Useful background for designing the custom
XML message format between `credora-backend` (Go) and the NIBSS simulator (Java).

---

## 1. What XML actually is

XML stores data as labeled, nested boxes. Instead of:

```
Griffin, 28, Lagos
```

XML labels each value so nothing has to be guessed:

```xml
<person>
  <name>Griffin</name>
  <age>28</age>
  <city>Lagos</city>
</person>
```

Each `<name>...</name>` pair is an **element**. Elements can nest inside other
elements to form a tree:

```xml
<person>
  <name>Griffin</name>
  <address>
    <city>Lagos</city>
    <country>Nigeria</country>
  </address>
</person>
```

Elements can also carry **attributes** — short descriptors pinned to the tag
itself rather than placed inside it:

```xml
<amount currency="NGN">5000</amount>
```

That's the entire vocabulary of XML: **elements, nesting, and attributes.**

### The problem XML alone doesn't solve

XML has no built-in way to stop garbage data — this is still "valid" XML even
though it's nonsense:

```xml
<person>
  <nam>Griffin</nam>
  <age>banana</age>
</person>
```

Tags open and close correctly, so XML itself is satisfied. Something else is
needed to say what "correct" data actually looks like.

---

## 2. What an XSD is for

An **XSD (XML Schema Definition)** is a separate rulebook document that
defines what a valid XML document of a given type must look like:

- what elements/attributes are allowed
- what order they go in
- how many times they can repeat
- what data types and constraints their values must satisfy

Once a schema exists, any XML-processing tool — in any language — can
automatically check a document against it and say "valid" or "invalid,"
without a human reading the content. This is exactly why two independent
systems (like a Go backend and a Java simulator) can trust messages passed
between them: both sides validate against the same shared contract.

XSD is itself written in XML — a schema file is just XML describing the shape
of other XML.

---

## 3. The core building blocks

### `xs:element` — declaring a labeled box

```xml
<xs:element name="age" type="xs:integer"/>
```
"There's a box called `age`, and it must hold a whole number."

### `minOccurs` / `maxOccurs` — how required / repeatable

Defaults are `minOccurs="1"` and `maxOccurs="1"` (exactly once, required).

```xml
<xs:element name="nickname" type="xs:string" minOccurs="0"/>
<xs:element name="phone" type="xs:string" maxOccurs="unbounded"/>
```
- `minOccurs="0"` → optional
- `maxOccurs="unbounded"` → repeatable, no limit (or use a specific number, e.g. `maxOccurs="3"`)

### `xs:sequence` — "all of these, in this order"

```xml
<xs:complexType name="PersonType">
  <xs:sequence>
    <xs:element name="name" type="xs:string"/>
    <xs:element name="age" type="xs:integer"/>
  </xs:sequence>
</xs:complexType>
```

### `xs:choice` — "one of these, not both"

```xml
<xs:complexType name="ContactMethod">
  <xs:choice>
    <xs:element name="email" type="xs:string"/>
    <xs:element name="phone" type="xs:string"/>
  </xs:choice>
</xs:complexType>
```
Valid XML has *either* `<email>` *or* `<phone>` — never both, never neither.

### `xs:enumeration` — a fixed list of allowed values

```xml
<xs:simpleType name="StatusType">
  <xs:restriction base="xs:string">
    <xs:enumeration value="active"/>
    <xs:enumeration value="suspended"/>
    <xs:enumeration value="closed"/>
  </xs:restriction>
</xs:simpleType>
```
Define the type once, reuse it anywhere via `type="StatusType"`.

### `xs:pattern` — a regex constraint instead of a fixed list

```xml
<xs:simpleType name="AccountNumberType">
  <xs:restriction base="xs:string">
    <xs:pattern value="[0-9]{10}"/>
  </xs:restriction>
</xs:simpleType>
```
"Exactly 10 digits, nothing else." (Same idea behind the IBAN pattern in the
ISO 20022 schema — just simpler.)

---

## 4. `simpleType` vs `complexType`

**One-sentence version:**
- `simpleType` = a box holding **one plain value only** — no child elements, no attributes.
- `complexType` = a box that can hold **other elements inside it** and/or **attributes on itself**.

"Simple" doesn't mean easy — it means "the content is just one plain value,"
not that it's inherently less useful.

**simpleType example:**
```xml
<xs:simpleType name="AgeType">
  <xs:restriction base="xs:integer">
    <xs:minInclusive value="0"/>
    <xs:maxInclusive value="120"/>
  </xs:restriction>
</xs:simpleType>
```
Valid content: `<age>28</age>` — nothing else allowed inside it.

**complexType example:**
```xml
<xs:complexType name="PersonType">
  <xs:sequence>
    <xs:element name="name" type="xs:string"/>
    <xs:element name="age" type="AgeType"/>
  </xs:sequence>
</xs:complexType>
```
Valid content:
```xml
<person>
  <name>Griffin</name>
  <age>28</age>
</person>
```
Child elements are only possible inside a `complexType` — a `simpleType`
physically cannot contain sub-boxes.

**The tricky middle case: a value WITH an attribute.**

```xml
<amount currency="NGN">5000</amount>
```
This looks "simple" (just a number) but it also carries an attribute, and
attributes are only legal on `complexType`. The fix is `simpleContent`:

```xml
<xs:complexType name="AmountType">
  <xs:simpleContent>
    <xs:extension base="xs:decimal">
      <xs:attribute name="currency" type="xs:string"/>
    </xs:extension>
  </xs:simpleContent>
</xs:complexType>
```
Read as: "complexType because it needs an attribute, but its inner content is
still just one plain value (`simpleContent`), a decimal, plus one attribute
(`currency`)." This exact pattern is `ActiveCurrencyAndAmount` in the ISO
20022 schema.

**Quick test when reading any XSD:** does this thing have child elements or
an attribute? Yes to either → `complexType`. No to both → `simpleType`.

---

## 5. XSD vs. Zod

Both do the same job — describing the shape of data so it can be validated
before being trusted — but they differ in important ways:

| | Zod | XSD |
|---|---|---|
| Lives | Inside your TypeScript code | A standalone `.xsd` file |
| Language-bound? | Yes, TypeScript/JS only | No — any language's XML tooling can read it |
| How you validate | Call `.parse()` directly in code | Point an XML library at the file; the library checks for you |
| Best fit | One codebase validating its own data | A contract shared between independent systems in different languages |

```typescript
// Zod
const PersonSchema = z.object({
  name: z.string(),
  age: z.number().int().min(0).max(120),
});
```
```xml
<!-- XSD, same idea -->
<xs:complexType name="PersonType">
  <xs:sequence>
    <xs:element name="name" type="xs:string"/>
    <xs:element name="age" type="AgeType"/>
  </xs:sequence>
</xs:complexType>
```

**Do you have to hand-write raw XSD like a Zod schema?** Not necessarily —
options for a custom, two-system format:
1. Hand-write the `.xsd` — reasonable for a scoped custom format.
2. Generate a starter XSD from sample XML documents, then refine.
3. Skip formal XSD validation entirely: define a Go struct (`encoding/xml`
   tags) and a matching Java class (JAXB/Jackson XML annotations) that agree
   by convention, with no shared `.xsd` file ever validated against.

A formal XSD adds value mainly for (a) strict validation independent of
either app's code, or (b) documentation of the format not tied to one
language. Real interbank standards (including NIP) use formal schemas for
exactly that reason — but for a two-system custom format, it's a deliberate
choice, not a requirement.

---

## 6. XML namespaces

### The problem namespaces solve

Imagine combining data from two different systems that both happen to use
the same tag names:

```xml
<!-- Company system -->
<employee><name>John</name></employee>

<!-- HR system -->
<employee><name>John</name></employee>
```

Both have `<employee>` and `<name>`, but they might mean entirely different
things. XML needs a way to say "this `<employee>` belongs to system A, that
one belongs to system B" — that's what a **namespace** is: a way to give a
name a globally unique identity so two vocabularies never collide.

### Declaring one with `xmlns`

```xml
<company:employee xmlns:company="https://example.com/company">
  <company:name>John</company:name>
</company:employee>
```

`xmlns:company="https://example.com/company"` means: *"whenever you see the
prefix `company:`, it refers to the vocabulary identified by
`https://example.com/company`."* So `company:employee` is really shorthand
for something like `{https://example.com/company}employee`.

**Important: the prefix (`company`) is not the namespace — it's just a local
alias pointing at the namespace.** These two documents are semantically
identical, because both prefixes point at the same URL:

```xml
<company:employee xmlns:company="https://example.com/company">
  <company:name>John</company:name>
</company:employee>
```
```xml
<c:employee xmlns:c="https://example.com/company">
  <c:name>John</c:name>
</c:employee>
```
What matters is `https://example.com/company` + `employee` — not the label
`company` or `c` used to abbreviate it.

### Why a URL?

URLs are just a convenient way to create a globally unique string — XML does
**not** make an HTTP request to that address. It's purely an identifier, the
same way a Java package name is:

```
com.mycompany.Employee        vs.        com.someothercompany.Employee
```

Both classes are called `Employee`, but their fully-qualified names differ.
XML namespaces do the same thing with URLs instead of dotted package paths:
`https://company.com + employee` is a different identity than
`https://hr.com + employee`.

**Mental model:** when you see `<foo:employee xmlns:foo="https://company.com">`,
read it as *"`employee`, from the `https://company.com` vocabulary."*

### Default namespaces (no prefix)

You don't always need a prefix — a **default namespace** applies to every
unprefixed tag in the document:

```xml
<employee xmlns="https://example.com/company">
  <name>John</name>
</employee>
```
This is equivalent in meaning to prefixing every tag with `company:`, without
having to actually write the prefix each time.

### Multiple vocabularies in one document

Real-world XML often mixes an envelope vocabulary with an application
vocabulary, e.g. SOAP:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header/>
  <soap:Body>
    <bank:Transfer xmlns:bank="https://example.com/banking">
      <bank:account>1234567890</bank:account>
      <bank:amount>3000</bank:amount>
    </bank:Transfer>
  </soap:Body>
</soap:Envelope>
```
`soap:Envelope`/`soap:Header`/`soap:Body` belong to the SOAP namespace;
`bank:Transfer`/`bank:account`/`bank:amount` belong to a separate banking
namespace — both living in the same document without colliding.

### Namespaces inside an XSD file itself

Since an XSD is written in XML, it needs its own namespace declarations at
the top. From the ISO 20022 `pain.001.001.13` schema:

```xml
<xs:schema 
    xmlns="urn:iso:std:iso:20022:tech:xsd:pain.001.001.13"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    elementFormDefault="qualified"
    targetNamespace="urn:iso:std:iso:20022:tech:xsd:pain.001.001.13">
```

- **`xmlns:xs="http://www.w3.org/2001/XMLSchema"`** — defines the `xs:`
  prefix as pointing to the official "language for writing schema rules."
  This is what allows `xs:element`, `xs:complexType`, `xs:sequence`, etc. to
  be used throughout the file — they're all vocabulary borrowed from that
  namespace, distinguishing "rule-writing tags" from the custom vocabulary
  being defined.
- **`xmlns="urn:iso:std:iso:20022:tech:xsd:pain.001.001.13"`** — the default
  namespace. Any unprefixed tag (like the type names `Document`,
  `PostalAddress27`, `Max35Text` being *defined*, as opposed to the `xs:`
  tags *doing the defining*) belongs to this ISO 20022 vocabulary.
- **`targetNamespace="urn:iso:std:iso:20022:tech:xsd:pain.001.001.13"`** —
  declares that everything defined in this file officially belongs to this
  namespace. Any real XML document claiming to follow this schema must
  declare the same namespace (usually via its own matching `xmlns=`). Note
  it's the identical string to the default `xmlns` above — that's what ties
  "the vocabulary I'm defining" to "the vocabulary this file claims to own."
- **`elementFormDefault="qualified"`** — a strictness setting: even nested
  child elements (not just top-level ones) must be understood as belonging
  to the target namespace in real XML instances. (`unqualified`, the
  default if omitted, is looser about nested elements.) Matters more for
  tooling correctness than for conceptual understanding.

Then:
```xml
<xs:element name="Document" type="Document"/>
```
- `xs:element` → rule-writing vocabulary: "I'm declaring a new element"
- `name="Document"` → its tag name
- `type="Document"` → its shape is defined by a `complexType` named
  `Document` elsewhere in the file — this is the **root element** declaration,
  i.e. the outermost box any valid document following this schema must have.

---

## 7. Applying this to a custom NIBSS message schema

Since `credora-backend` (Go) and the NIBSS simulator (Java) are two systems
you fully control, a formal XSD isn't strictly required — but if writing one
for learning/production-realism, useful patterns to borrow from ISO 20022
rather than copy wholesale:

- **Reusable primitive types** (like `Max35Text`) instead of repeating
  length/pattern constraints inline every time.
- **A payment-identification pattern** (something like an instruction ID +
  end-to-end ID + a UUID-style unique reference) so both systems can
  correlate a transfer across systems and retries.
- **A bank/agent identity type** (identifier + name + address) for modeling
  both the originating side (Monnify-like) and the receiving side
  (Moniepoint-like).
- **Closed enums** (`xs:enumeration`) for small fixed sets of states —
  transfer status, charge bearer, etc. — instead of trusting free-text
  strings.
- A dedicated namespace, e.g. `urn:credora:nibss:v1`, instead of reusing
  ISO's.

Everything specific to cheques, mandates, tax reporting, and regulatory
reporting in the ISO 20022 file can be skipped — there's no NIP/interbank-rail
equivalent for a scoped custom format.
