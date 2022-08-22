First pass with everything. Not friendly when either import is not there

```js
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  let node = root.find(j.ImportDeclaration)
    .filter(p => p.value.source.value === 'components/elements/redirect')

  if (!node) {
    console.log('exit early')
    return root.toSource();
  }

  const localName = node.find(j.ImportDefaultSpecifier).get().value.local.name

  let rrd = root.find(j.ImportDeclaration)
    .filter(p => p.value.source.value === 'react-router-dom')
    .get();

  // if react-router-dom is already imported
  if (rrd) {
    // remove the old import
    node.remove();
    // if `Redirect` is not already in the import, add it
    if (!rrd.value.specifiers.some(spec => spec.imported.name === 'Redirect')) {
      rrd.value.specifiers.push(j.importSpecifier(j.identifier('Redirect')))
    }
  }
  else {
    // if no rrd import just replace the old import
    importDeclaration.get().replace("import { Redirect } from 'react-router-dom';")
  }

  // Refactor: replace property is the opposite of push property
  root.findJSXElements(localName).forEach(path => {
    path.value.openingElement.name.name = 'Redirect';
    let { attributes } = path.value.openingElement
    const replace = attributes.find(attr => attr.name.name === 'replace')
    if (replace) {
      path.value.openingElement.attributes = attributes.filter(attr => attr.name.name !== 'replace')
    } else {
      attributes.push(j.jsxAttribute(j.jsxIdentifier('push')));
    }
  });

  return root.toSource();
};
```

use typing to help, simplify some filters, empty friendly

```js
import core from '@types/jscodeshift';

/**
 * @param file {core.FileInfo}
 * @param api {core.API}
 **/
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  let node = root.find(j.ImportDeclaration, {source: {value: 'components/elements/redirect'}})

  if (!node.length) {
    return root.toSource();
  }

  /** @type { core.ASTPath }*/
  const got = node.find(j.ImportDefaultSpecifier).get();

  const localName = node.find(j.ImportDefaultSpecifier).get('local', 'name').value

  let rrd = root.find(j.ImportDeclaration, {source: {value: 'react-router-dom'}});
  if (rrd.length) rrd = rrd.get();

  // if react-router-dom is already imported
  if (rrd) {
    // remove the old import
    node.remove();
    // if `Redirect` is not already in the import, add it
    if (!rrd.value.specifiers.some(spec => spec.imported.name === 'Redirect')) {
      rrd.value.specifiers.push(j.importSpecifier(j.identifier('Redirect')))
    }
  }
  else {
    // if no rrd import just replace the old import
    importDeclaration.get().replace("import { Redirect } from 'react-router-dom';")
  }

  // Refactor: replace property is the opposite of push property
  root.findJSXElements(localName).forEach(path => {
    path.value.openingElement.name.name = 'Redirect';
    let { attributes } = path.value.openingElement
    const replace = attributes.find(attr => attr.name.name === 'replace')
    if (replace) {
      path.value.openingElement.attributes = attributes.filter(attr => attr.name.name !== 'replace')
    } else {
      attributes.push(j.jsxAttribute(j.jsxIdentifier('push')));
    }
  });

  return root.toSource();
};
```

final
```
// Commands to run (must be run in higher bash version than default on mac to get ** glob to work)
// find frontend/javascripts/components/ -name *.[jt]sx | xargs jscodeshift -t script/rr-redirect-migration.js --parser=tsx
// find frontend/javascripts/spec/components/ -name *.[jt]sx | xargs jscodeshift -t script/rr-redirect-migration.js --parser=tsx

/** @typedef { import('@types/jscodeshift').core } core */
/**
 * @param file {core.FileInfo}
 * @param api {core.API}
 **/
export default function transformer(file, api) {
  const j = api.jscodeshift;
  let root = j(file.source);

  let importDeclaration = root.find(j.ImportDeclaration, {source: {value: 'components/elements/redirect'}})

  if (!importDeclaration.length) {
    return root.toSource();
  }

  const localName = importDeclaration.find(j.ImportDefaultSpecifier).get('local', 'name').value

  let rrdImport = root.find(j.ImportDeclaration, {source: {value: 'react-router-dom'}});
  // if react-router-dom is already imported add the named import `Redirect` if not already there
  if (rrdImport.length) {
    let importNodeValue = rrdImport.get().value
    // if `Redirect` is not already there, add it
    if (!importNodeValue.specifiers.some(s => s.imported.name === 'Redirect')) {
      importNodeValue.specifiers.push(j.importSpecifier(j.identifier('Redirect')))
    }
    // remove the old import
    importDeclaration.remove();
  }
  else {
    // if react-router-dom was not imported, add it, replacing the old import
    importDeclaration.get().replace("import { Redirect } from 'react-router-dom';")
  }

  // Now, do the migration from one component to the other
  root.findJSXElements(localName).forEach(path => {
    path.value.openingElement.name.name = 'Redirect';

    // The `replace` prop is the opposite of `push` property
    let { attributes } = path.value.openingElement
    const hasReplaceProp = attributes.find(attr => attr.name.name === 'replace')
    if (hasReplaceProp) {
      path.value.openingElement.attributes = attributes.filter(attr => attr.name.name !== 'replace')
    } else {
      attributes.push(j.jsxAttribute(j.jsxIdentifier('push')));
    }
  });

  return root.toSource();
};

```