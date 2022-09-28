# Custom Import Handler

## Status

**Stage:** 0

**Author:** [JOTSR](https://github.com/JOTSR)

**Champions:** [JOTSR](https://github.com/JOTSR)

## Overview / Motivation

Currently there is no standard way to import custom assets in JS, however it is widely used by transpilers. Expecting standardizations and implementations can be long, messy, and cause many compatibilities issues. In order to allow users to extends imports types independently of implementers and specifiers this proposal provide a standardization of custom imports directly for users or tools like babel, esbuild, and helpful hand for LSPs. So, this provides a native way to bundle assets for example for CSR with React, .... Inspired by [JSON modules proposal](https://github.com/tc39/proposal-json-modules) and as an extension to [Import assertions proposal](https://github.com/tc39/proposal-import-assertions). It encompasses [asset references porposal](https://github.com/tc39/proposal-asset-references).

## API

There are two suggested APIs based on simplicity and security constraints.

### Handler Module Blocks API

An handler can be passed to import assert as a path to a module or directly a module block (see [JS Module Blocks proposal](https://github.com/tc39/proposal-js-module-blocks))

**Pros**
+ No side effects (provide host isolation from handler)
+ Can be resolve before host runtime

**Cons**
- Handler can't be included in a bundle
- Only one handler is authorized by files since it is mandatory export as default (one handler can support mutiple types)

```ts
type HandlerModule = Module {
    default: (source: Uint8Array, { url, type, mimeType, encoding, nativeHandler } : { url: URL, type: string, mimeType: string, encoding: 'blob' | encodingType, nativeHandler: boolean }) => Module
}

interface Module {
    default?: unknown
    [exported: string]: unknown
}

//Static imports
import defaultExport from '' assert { type: string, handler: HandlerModule | 'handlerModulePath' }
import {} from '' assert { type: string, handler: HandlerModule | 'handlerModulePath' }
import defaultExport, {} from '' assert { type: string, handler: HandlerModule | 'handlerModulePath' }
import defaultExport, * as {} from '' assert { type: string, handler: HandlerModule | 'handlerModulePath' }
import * as {} from '' assert { type: string, handler: HandlerModule | 'handlerModulePath' }
import '' assert { type: string, handler: HandlerModule | 'handlerModulePath' } //Simple import syntax not allowed

//Dynamic import
import('', { assert: { type: string, handler: HandlerModule | 'handlerModulePath' }})
```

The handler takes the source as Uint8Array in order to allow use of non textual imports, like pictures, librairies for runtimes like Deno or Node, ...
The handler get the url of the import, the asserted type, the resolved mime-type and encoding.
The nativeHandler key indicate if the runtime already support the asserted type, following impletation choice it can be use as fallback if overriding is avoid or leave the choice to use native handler for better performances.

The handler return an object that mimic a standard es module i.e. a js module encapsulating the imports datas.

The initial behaviour is to run the module during the import resolution in order to not impact the host script runtime. The cache should use the returned module if the import and the handler are unchanched.

### Function Handler API

This design use a function instead of a module for handling custom type import.

**Pros**
+ Simple API

**Cons**
- Possible side effect since JS can't verify functions purity

```ts
type Handler = (source: Uint8Array, { url: URL, type: string, mimeType: string, encoding: 'blob' | encodingType, nativeHandler: boolean }) => Module

interface Module {
    default?: unknown,
    [string]: unknown
}

//Static imports
import defaultExport from '' assert { type: string, handler: Handler }
import {} from '' assert { type: string, handler: Handler }
import defaultExport, {} from '' assert { type: string, handler: Handler }
import defaultExport, * as {} from '' assert { type: string, handler: Handler }
import * as {} from '' assert { type: string, handler: Handler }
import '' assert { type: string, handler: Handler } //Simple import syntax not allowed

//Dynamic import
import('', { assert: { type: string, handler: Handler }})
```

### Optional: Native Handler Override API

```ts 
//Handler or HandlerModule | "handlerModulePath" depending on Handler implemention choice

interface CustomImportHandler {
    define: ({ types, handler }: { types: string[], handler: Handler }) => void
    for: (type: string) => { types: string[], handler: Handler }
    entries: () => Iterator<{ types: string[], handler: Handler }>
    nativeFor: (type: string) => boolean
}
```

The CustomImportHandler is a global scope object that provides a way to assign a type to a default handler. The define method declare a default handler that can be overrided with the "hadler" key in import assert.
The for method return the default handler for a type and the types linked of the handler.
Entries return an iterator to loop over handlers.
nativeFor return the existence of a native runtime handler for the specified type.

## Examples

### CSV file
```ts
//main.js
import * as csvDatas from './datas.csv' assert { type: 'csv', handler: './csvImportsHandler.js' }

//csvImportsHandler.js
import { parse } from './csvParser.js'

export default function handler(source,  { url, type, mimeType, encoding, nativeHandler }) {
    const text = new TextDecoder(encoding).decode(source)
    return {
        default: parse(text)
    }
}
```

### JSON with comment support file
```ts
//main.js
import { users, config } from './datas.jsonc' assert { type: 'jsonc', handler: './importsHandler.js' }

//csvImportsHandler.js
import { parse as csvParse } from './csvParser.js'
import { parse as jsoncParse } from './jsoncParser.js'
import { parse as yamlParse } from './yamlParser.js'

//multiple type handling
export default function handler(source,  { url, type, mimeType, encoding, nativeHandler }) {
    const text = new TextDecoder(encoding).decode(source)
    if (type === 'jsonc') {
        return {
            default: jsoncParse(text)
        }
    }
    if (type === 'yaml') {
        return {
            default: yamlParse(text)
        }
    }
    if (type === 'csv') {
        return {
            default: csvParse(text)
        }
    }

    throw new TypeError(`Current handler can't parse import of type ${type}`)
}
```

### Assets support
```ts
//main.js
import { image as logo } from './logo.png' assert { type: 'png', handler: './assetsImportsHandler.js' }

//csvImportsHandler.js
import { parse } from './csvParser.js'

export default function handler(source,  { url, type, mimeType, encoding, nativeHandler }) {
    if (!['image/png', 'image/jpeg', 'image/bmp'].includes(mimeType)) throw new TypeError(`Current handler can't decode the mime-type ${mimeType}`)

    const image = new Image()
    image.src = url.toString()

    return {
        image,
        url,
        mimeType
    }
}
```

### Native Handler Override API
```ts 
//main.js
if (CustomImportHandler.nativeFor('css') && CustomImportHandler.for('css')) {
    CustomImportHandler.define({ types: ['scss', 'css', 'sass'], handler: './path/to/cssImportHandler' })
}
```

## FAQs

- Should type property must use custom notation, mimetype notation, or a mixed ?
- Sould datas passed in stream instead of Uint8Array or allow string (possibly optimisation issues) ?
- Handler declared in assert field, in a new field or with a new keyword ?
- Should handler return a module block instead of an object, how to initialize it ?
- When imports is made, during parse, during runtime, dureing runtime only with dynamic imports, can be realized outside of host runtime with "Hadler function API" implementation ?
- Allow to handle/override js/ts imports (for eg: transpiling) ?
- Allow to hadle imports without specifiate type ?
- Can an export use an handler as proxy ?
- Override API conflict with libraries ? How to define priority ?
