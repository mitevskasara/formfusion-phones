# @formfusion/phones

Set of validation rules for worldwide phone numbers, used for the [FormFusion](https://www.corelabui.com/formfusion) (form management & validation) library.

A zero-dependency lookup table of **155 country-specific regex patterns** for validating phone numbers. Every value is a regex **string** (no `null`s, no empty placeholders) and works directly as an HTML [`pattern`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern) attribute value, so you can use it with plain HTML, React, FormFusion, or `new RegExp()`.

## Why

Phone number formats differ per country. Germany requires a `+49` or `0` prefix and a `1` service digit, the UK is a fixed `+44 7` mobile shape, Brazil mixes optional area-code parentheses and dashes, Indonesia accepts spaces or nothing between subscriber groups, and Japan allows `-` separators in two places. Every app that collects a phone number ends up re-implementing and re-maintaining this table.

This package ships it as one flat object so you don't have to.

## Installation

```bash
npm install @formfusion/phones
```

```bash
yarn add @formfusion/phones
```

## Usage

The package exports a single default object mapping lowercase ISO 3166-1 alpha-2 country codes to regex **strings**.

### ES modules

```js
import phones from '@formfusion/phones';

console.log(phones.gb); // "^(\\+?44|0)7\\d{9}$"
```

### CommonJS

```js
const phones = require('@formfusion/phones').default;

new RegExp(phones.gb).test('+447123456789'); // true
new RegExp(phones.gb).test('12345'); // false
```

### Plain HTML

The patterns are valid `pattern` attribute values, so they work without any JavaScript:

```html
<label for="phone">Mobile number (United Kingdom)</label>
<input id="phone" name="phone" type="tel" pattern="^(\+?44|0)7\d{9}$" required />
```

### With FormFusion

FormFusion passes unknown `type` values straight through to the input's `pattern` attribute, so you can hand it a pattern directly:

```jsx
import React from 'react';
import { Form, Input } from 'formfusion';
import 'formfusion/style.css';
import phones from '@formfusion/phones';

const MyForm = () => (
  <Form onSubmit={(data) => console.log('Submitted', data)}>
    <Input id="phone" name="phone" type={phones.de} label="Phone number" required />
    <button type="submit">Submit</button>
  </Form>
);
```

Patterns compose with FormFusion's `rules` combinators if you need to accept more than one country:

```jsx
import { Input, rules } from 'formfusion';
import phones from '@formfusion/phones';

// Accept either a German or an Austrian number
<Input name="phone" type={rules.existIn([phones.de, phones.at])} />
```

### Dynamic country selection

```jsx
const [country, setCountry] = useState('de');

<Select name="country" value={country} onChange={setCountry}>
  {Object.keys(phones).map((code) => (
    <option key={code} value={code}>
      {code.toUpperCase()}
    </option>
  ))}
</Select>

<Input name="phone" type={phones[country] || 'text'} label="Phone number" required />
```

### Standalone validation

```js
import phones from '@formfusion/phones';

export function isValidPhone(value, country) {
  const pattern = phones[String(country).toLowerCase()];
  if (!pattern) return false; // unknown country
  return new RegExp(pattern).test(value);
}

isValidPhone('+447123456789', 'GB'); // true
isValidPhone('12345', 'gb'); // false
```

### TypeScript

Typings are hand-written in `index.d.ts` and mirror the runtime exactly — all **155 keys** are declared, uppercase in the `Phones` type and lowercased via a mapped type (`LowercaseKeys`) so the exported keys match the runtime object. Because the declaration uses `export =`, you need `esModuleInterop` or `allowSyntheticDefaultImports`.

```ts
import phones from '@formfusion/phones';

const gb: string = phones.gb;
// @ts-expect-error - unknown country
const xx: string = phones.xx;
```

## API

The export is a plain object with no functions or classes:

```ts
{ [countryCode: string]: string }
```

Country codes are **lowercase** (`de`, `gb`, `br`). Lookups are case-sensitive, so normalize user input first.

Every entry is a string — unlike `@formfusion/vat`, there are no `null` entries and no empty placeholder patterns. Enumerate the available codes with `Object.keys(phones)`.

**Prefix handling varies by pattern.** Phone numbers are usually written with a country code, so most patterns include a `+`-style prefix (`(\+?44|0)`, `((\+?20)|0)?`, …) and some also accept the `00` international dialing prefix (e.g. `ma`, `om`, `cn`). The prefix is **optional in most patterns but required in a few** (e.g. `kw` requires `965`, `ni` requires `505`), and the accepted style (`+`, `+?`, `0`, `00`, bare) differs per country. Test the specific country you rely on rather than assuming E.164 behavior.

Coverage: 155 countries and territories across all regions.

## Caveats

Read these before relying on the patterns.

**Format only, no checksum.** These are shape checks. There is no libphonenumber-style validation, no carrier lookup, and no E.164 normalization. A well-formed number that does not exist will pass. For authoritative verification use a dedicated phone library or a carrier/number-lookup service.

**Prefix expectations vary.** As noted above, some patterns reject numbers without a country code, others accept a bare national number, and the accepted separators (`-`, space, none) differ. Normalize input (strip spaces/dashes) before testing if users may paste in different styles — but note that some patterns (`br`, `jp`, `id`) explicitly allow separators, so aggressive stripping can also break them.

**Partial anchoring in `am` and `bm`.** These two patterns place `$` inside their alternations rather than once at the end (`^(…\d{6}$|[2-4]\d{7}$)`), so with `new RegExp(...)` some branches can be followed by extra characters and still match. When used in an HTML `pattern` attribute, the browser implicitly anchors the whole value as `^(?:…)$`, which mitigates this.

**Keys are lowercased at runtime.** `index.d.ts` declares uppercase keys (`GB`, `DE`) and maps them to lowercase for the exported type; the built `index.js` exposes lowercase keys only.

## Development

```bash
git clone https://github.com/mitevskasara/formfusion-phones.git
cd formfusion-phones
npm install
npm run build
```

### How it works

All source lives in [`src/index.js`](src/index.js) as a single object of uppercase country codes. The last step lowercases every key before exporting, so `GB` becomes `gb`.

[`esbuild.js`](esbuild.js) bundles that into a minified CommonJS `index.js` at the repo root, targeting Node 14. Consumers get the built file, so **changes are not live until you rebuild and commit `index.js`**:

```bash
npm run build
```

The source currently contains a commented-out duplicate `IR` entry (an older pattern variant) next to the live one; the built `index.js` has 155 unique keys with `ir` appearing exactly once.

This repo has no tests and no CI. If you add a pattern, add a corresponding test in the [FormFusion](https://github.com/corelabui/formfusion) repo, which covers the sibling packages with one Jest + React Testing Library file per country.

### Scripts

| Script | Description |
| --- | --- |
| `npm run build` | Clean stale build output, then bundle `src/index.js` into `index.js` via esbuild |
| `npm version <patch\|minor\|major>` | Bump the version and regenerate `CHANGELOG.md` from Conventional Commits (runs `npm run version` automatically) |
| `npm run publish-package` | `npm publish --access public` |

The `version` script shells out to `conventional-changelog`, which is not declared in `devDependencies`. Install it globally or add it as a dev dependency before running a version bump.

### Commit convention

This repo follows [Conventional Commits](https://www.conventionalcommits.org/), and `CHANGELOG.md` is generated from those subjects:

```
Feat: add Nepali phone pattern
Fix: make the UK prefix optional
```

### Adding a country

1. Add the entry to `src/index.js`, using an uppercase country code.
2. Add the uppercase key to the `Phones` type in `index.d.ts`.
3. Run `npm run build` and commit the regenerated `index.js`.
4. Add a test in the FormFusion repo under `src/__tests__/phones/`.

## Related packages

Part of the FormFusion family of extracted validation rule sets:

- [`@formfusion/postcodes`](https://www.npmjs.com/package/@formfusion/postcodes)
- [`@formfusion/licence-plates`](https://www.npmjs.com/package/@formfusion/licence-plates)
- [`@formfusion/iban`](https://www.npmjs.com/package/@formfusion/iban)
- [`@formfusion/passports`](https://www.npmjs.com/package/@formfusion/passports)
- [`@formfusion/vat`](https://www.npmjs.com/package/@formfusion/vat)
- [`@formfusion/tin`](https://www.npmjs.com/package/@formfusion/tin)
- [`formfusion`](https://www.npmjs.com/package/formfusion) — the core library

## Issues

Report bugs and feature requests at https://github.com/mitevskasara/formfusion-phones/issues.

## License

BSD-2-Clause. Copyright (c) 2023, Mitevska Sara.
