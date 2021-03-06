# Docx-templates [![Build Status](https://travis-ci.org/guigrpa/docx-templates.svg)](https://travis-ci.org/guigrpa/docx-templates) [![Coverage Status](https://coveralls.io/repos/github/guigrpa/docx-templates/badge.svg?branch=master)](https://coveralls.io/github/guigrpa/docx-templates?branch=master) [![npm version](https://img.shields.io/npm/v/docx-templates.svg)](https://www.npmjs.com/package/docx-templates)

Template-based docx report creation for both Node and the browser. ([See the blog post](http://guigrpa.github.io/2017/01/01/word-docs-the-relay-way/)).


## Why?

* **Write documents naturally using Word**, just adding some commands where needed for dynamic contents
* **Express your data needs (queries) in the template itself** (`QUERY` command), in whatever query language you want (e.g. in GraphQL). This is similar to _the Relay way™_: in [Relay](https://facebook.github.io/relay/), data requirements are declared alongside the React components that need the data
* **Execute JavaScript snippets** (`EXEC` command, or `!` for short)
* **Insert the result of JavaScript snippets** in your document (`INS`, `=` or just *nothing*)
* **Embed images, hyperlinks and even HTML dynamically** (`IMAGE`, `LINK`, `HTML`). Dynamic images can be great for on-the-fly QR codes, downloading photos straight to your reports, charts… even maps!
* Add **loops** with `FOR`/`END-FOR` commands, with support for table rows, nested loops, and JavaScript processing of elements (filter, sort, etc)
* Include contents **conditionally**, `IF` a certain JavaScript expression is truthy
* Define custom **aliases** for some commands (`ALIAS`) — useful for writing table templates!
* Run all JavaScript in a **separate Node VM for security**
* Include **literal XML**
* Written in TypeScript, so ships with type definitions.
* Plenty of **examples** in this repo (with Node, Webpack and Browserify)

Contributions are welcome!


## Installation

```
$ npm install docx-templates
```

...or using yarn:

```
$ yarn add docx-templates
```


## Node usage

Here is a simple example, with report data injected directly as an object:

```js
import createReport from 'docx-templates';
import fs from 'fs';

const template = fs.readFileSync('myTemplate.docx');

const buffer = await createReport({
  template,
  data: {
    name: 'John',
    surname: 'Appleseed',
  },
});

fs.writeFileSync('report.docx', buffer)
```

You can also **provide a sync or Promise-returning callback function (query resolver)** instead of a `data` object:

```js
const report = await createReport({
  template,
  data: query => graphqlServer.execute(query),
});
```

Your resolver callback will receive the query embedded in the template (in a `QUERY` command) as an argument.

Other options (with defaults):

```js
const report = await createReport({
  // ...
  additionalJsContext: {
    // all of these will be available to JS snippets in your template commands (see below)
    foo: 'bar',
    qrCode: async url => {
      /* build QR and return image data */
    },
  },
  cmdDelimiter: '+++',
    /* default for backwards compatibility; but even better: ['{', '}'] */
  literalXmlDelimiter: '||',
  processLineBreaks: true,
  noSandbox: false,
});
```

Check out the [Node examples folder](https://github.com/guigrpa/docx-templates/tree/master/packages/example-node).


## Browser usage

You can use docx-templates in the browser (yay!). Just as when using docx-templates in Node, you need to provide the template contents as a buffer-like object. For example, you can get a `File` object with:

```html
<input type="file">
```

Then read this file in an ArrayBuffer, feed it to docx-templates, and download the result:

```js
import createReport from 'docx-templates';

const onTemplateChosen = async () => {
  const template = await readFileIntoArrayBuffer(myFile);
  const report = await createReport({
    template,
    data: { name: 'John', surname: 'Appleseed' },
  });
  saveDataToFile(
    report,
    'report.docx',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
  );
};

// Load the user-provided file into an ArrayBuffer
const readFileIntoArrayBuffer = fd =>
  new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onerror = reject;
    reader.onload = () => {
      resolve(reader.result);
    };
    reader.readAsArrayBuffer(fd);
  });
```

You can find an example implementation of `saveDataToFile()` [in the Webpack example](https://github.com/guigrpa/docx-templates/blob/79119723ff1c009b5bbdd28016558da9b405742f/examples/example-webpack/client/index.js#L82).

With the default configuration, browser usage can become slow with complex templates due to the usage of JS sandboxes for security reasons. _If the templates you'll be using with docx-templates can be trusted 100%, you can disable the security sandboxes by configuring `noSandbox: true`_. **Beware of arbitrary code injection risks**:

```js
const report = await createReport({
  // ...
  // USE ONLY IN THE BROWSER, AND WITH TRUSTED TEMPLATES
  noSandbox: true, // WARNING: INSECURE
});
```

Check out the examples [using Webpack](https://github.com/guigrpa/docx-templates/tree/master/packages/example-webpack) and [using Browserify](https://github.com/guigrpa/docx-templates/tree/master/packages/example-browserify).

### Browser compatibility caveat
Note that the JavaScript code in your docx template will be run as-is by the browser. Transpilers like Babel can't see this code, and won't be able to polyfill it. This means that the JS code in your template needs to be compatible with the browsers you are targeting. In other words: don't use fancy modern syntax and functions in your template if you want older browsers, like IE11, to be able to render it.

## Custom command delimiters
You can use different **left/right command delimiters** by passing an array to `cmdDelimiter`:

```js
const report = await createReport({
  // ...
  cmdDelimiter: ['{', '}'],
})
```

This allows much cleaner-looking templates!

Then you can add commands and JS snippets in your template like this: `{foo}`, `{project.name}` `{QUERY ...}`, `{FOR ...}`.

When choosing a delimiter, take care not to introduce conflicts with JS syntax, especially if you are planning to use larger JS code snippets in your templates. For example, with `['{', '}']` you may run into conflicts as the brackets in your JS code may be mistaken for command delimiters. As an alternative, consider using multi-character delimiters, like `{#` and `#}` (see issue [#102](https://github.com/guigrpa/docx-templates/issues/102)).

## Writing templates

You can find several template examples in this repo:

* [SWAPI](https://github.com/guigrpa/docx-templates/tree/master/packages/example-node), a good example of what you can achieve embedding a template (GraphQL in this case) in your report, including a simple script for report generation. Uses the freak-ish online [Star Wars GraphQL API](https://github.com/graphql/swapi-graphql).
* [Dynamic images](https://github.com/guigrpa/docx-templates/tree/master/packages/example-node): with examples of images that are dynamically downloaded or created. Check out the _images-many-tiles_ example for a taste of this powerful feature.
* Browser-based examples [using Webpack](https://github.com/guigrpa/docx-templates/tree/master/packages/example-webpack) and [using Browserify](https://github.com/guigrpa/docx-templates/tree/master/packages/example-browserify).

Currently supported commands are defined below.

### `QUERY`

You can use GraphQL, SQL, whatever you want: the query will be passed unchanged to your `data` query resolver.

```
+++QUERY
query getData($projectId: Int!) {
  project(id: $projectId) {
    name
    details { year }
    people(sortedBy: "name") { name }
  }
}
+++
```

For the following sections (except where noted), we assume the following dataset:

```js
const data = {
  project: {
    name: 'docx-templates',
    details: { year: '2016' },
    people: [{ name: 'John', since: 2015 }, { name: 'Robert', since: 2010 }],
  },
};
```

### `INS` (`=`, or nothing at all)

Inserts the result of a given JavaScript snippet:

```
+++INS project.name+++ (+++INS project.details.year+++)
or...
+++INS `${project.name} (${$details.year})`+++
```

Note that the last evaluated expression is inserted into the document, so you can include more complex code if you wish:

```
+++INS
const a = Math.random();
const b = Math.round((a - 0.5) * 20);
`A number between -10 and 10: ${b}.`
+++
```

You can also use this shorthand notation:

```
+++= project.name+++ (+++= project.details.year+++)
+++= `${project.name} (${$details.year})`+++
```

Even shorter (and with custom `cmdDelimiter: ['{', '}']`):

```
{project.name} ({project.details.year})
```

You can also access functions in the `additionalJsContext` parameter to `createReport()`, which may even return a Promise. The resolved value of the Promise will be inserted in the document.

Use JavaScript's ternary operator to implement an _if-else_ structure:

```
+++= $details.year != null ? `(${$details.year})` : ''+++
```

### `EXEC` (`!`)

Executes a given JavaScript snippet, just like `INS` or `=`, but doesn't insert anything in the document. You can use `EXEC`, for example, to define functions or constants before using them elsewhere in your template.

```
+++EXEC
myFun = () => Math.random();
MY_CONSTANT = 3;
+++

+++! ANOTHER_CONSTANT = 5; +++
```

Usage elsewhere will then look like

```
+++= MY_CONSTANT +++
+++= ANOTHER_CONSTANT +++
+++= myFun() +++
```

### `IMAGE`

Includes an image with the data resulting from evaluating a JavaScript snippet:

```
+++IMAGE qrCode(project.url)+++
```

In this case, we use a function from `additionalJsContext` object passed to `createReport()` that looks like this:

```js
  additionalJsContext: {
    qrCode: url => {
      const dataUrl = createQrImage(url, { size: 500 });
      const data = dataUrl.slice('data:image/gif;base64,'.length);
      return { width: 6, height: 6, data, extension: '.gif' };
    },
  }
```

The JS snippet must return an _image object_ or a Promise of an _image object_, containing:

* `width`: desired width of the image on the page _in cm_. Note that the aspect ratio should match that of the input image to avoid stretching.
* `height` desired height of the image on the page _in cm_.
* `data`: either an ArrayBuffer or a base64 string with the image data
* `extension`: e.g. `.png`
* `thumbnail` _[optional]_: when injecting an SVG image, a fallback non-SVG (png/jpg/gif, etc.) image can be provided. This thumbnail is used when SVG images are not supported (e.g. older versions of Word) or when the document is previewed by e.g. Windows Explorer. See usage example below.

In the .docx template:
```
+++IMAGE injectSvg()+++
```

In the `createReport` call:
```js
additionalJsContext: {
  injectSvg: () => {
      const svg_data = Buffer.from(`<svg  xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
                                  <rect x="10" y="10" height="100" width="100" style="stroke:#ff0000; fill: #0000ff"/>
                                </svg>`, 'utf-8');

      // Providing a thumbnail is technically optional, as newer versions of Word will just ignore it.
      const thumbnail = {
        data: fs.readFileSync('sample.png'),
        extension: '.png',
      };
      return { width: 6, height: 6, data: svg_data, extension: '.svg', thumbnail };                    
    }
  }
```

### `LINK`

Includes a hyperlink with the data resulting from evaluating a JavaScript snippet:

```
+++LINK ({ url: project.url, label: project.name })+++
```

If the `label` is not specified, the URL is used as a label.

### `HTML`

Takes the HTML resulting from evaluating a JavaScript snippet and converts it to Word contents (using [altchunk](https://blogs.msdn.microsoft.com/ericwhite/2008/10/26/how-to-use-altchunk-for-document-assembly/)):

```
+++HTML `
<body>
  <h1>${$film.title}</h1>
  <h3>${$film.releaseDate.slice(0, 4)}</h3>
  <p>
    <strong style="color: red;">This paragraph should be red and strong</strong>
  </p>
</body>
`+++
```

### `FOR` and `END-FOR`

Loop over a group of elements (resulting from the evaluation of a JavaScript expression):

```
+++FOR person IN project.people+++
+++INS $person.name+++ (since +++INS $person.since+++)
+++END-FOR person+++
```

Since JavaScript expressions are supported, you can for example filter the loop domain:

```
+++FOR person IN project.people.filter(person => person.since > 2013)+++
...
```

`FOR` loops also work over table rows:

```
----------------------------------------------------------
| Name                         | Since                   |
----------------------------------------------------------
| +++FOR person IN             |                         |
| project.people+++            |                         |
----------------------------------------------------------
| +++INS $person.name+++       | +++INS $person.since+++ |
----------------------------------------------------------
| +++END-FOR person+++         |                         |
----------------------------------------------------------
```

Finally, you can nest loops (this example assumes a different data set):

```
+++FOR company IN companies+++
+++INS $company.name+++
+++FOR person IN $company.people+++
* +++INS $person.firstName+++
+++FOR project IN $person.projects+++
    - +++INS $project.name+++
+++END-FOR project+++
+++END-FOR person+++

+++END-FOR company+++
```

### `IF` and `END-IF`

Include contents conditionally (depending on the evaluation of a JavaScript expression):

```
+++IF person.name === 'Guillermo'+++
+++= person.fullName +++
+++END-IF+++
```

Similarly to the `FOR` command, it also works over table rows. You can also nest `IF` commands
and mix & match `IF` and `FOR` commands. In fact, for the technically inclined: the `IF` command
is implemented as a `FOR` command with 1 or 0 iterations, depending on the expression value.

### `ALIAS` (and alias resolution with `*`)

Define a name for a complete command (especially useful for formatting tables):

```
+++ALIAS name INS $person.name+++
+++ALIAS since INS $person.since+++

----------------------------------------------------------
| Name                         | Since                   |
----------------------------------------------------------
| +++FOR person IN             |                         |
| project.people+++            |                         |
----------------------------------------------------------
| +++*name+++                  | +++*since+++            |
----------------------------------------------------------
| +++END-FOR person+++         |                         |
----------------------------------------------------------
```

## Error handling

By default, the Promise returned by `createReport` will reject with an error immediately when a problem is encountered in the template, such as a bad command (i.e. it 'fails fast'). In some cases, however, you may want to collect all errors that may exist in the template before failing. For example, this is useful when you are letting your users create templates interactively. You can disable fast-failing by providing the `failFast: false` parameter as shown below. This will make `createReport` reject with an array of errors instead of a single error so you can get a more complete picture of what is wrong with the template.

```typescript
try {
  const report = await createReport({
    template,
    data: {
      name: 'John',
      surname: 'Appleseed',
    },
    failFast: false,
  });
} catch (errors) {
  if (Array.isArray(errors)) {
    // An array of errors likely caused by bad commands in the template.
    console.log(errors);
  } else {
    // Not an array of template errors, indicating something more serious.
    throw errors;
  }
}
```


## [Changelog](https://github.com/guigrpa/docx-templates/blob/master/CHANGELOG.md)


## Similar projects

* [docxtemplater](https://github.com/open-xml-templating/docxtemplater) (believe it or not, I just discovered this very similarly-named project after brushing up my old CS code for `docx-templates` and publishing it for the first time!). It provides lots of goodies, but doesn't allow (AFAIK) embedding queries or JS snippets.

* [docx](https://github.com/dolanmiu/docx) and similar ones - generate docx files from scratch, programmatically. Drawbacks of this approach: they typically do not support all Word features, and producing a complex document can be challenging.


## License (MIT)

Copyright (c) [Guillermo Grau Panea](https://github.com/guigrpa) 2016-now

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
