<h1 align="center">
  <p align="center">Docusaurus</p>
  <a href="https://docusaurus.io"><img src="https://docusaurus.io/img/slash-introducing.svg" alt="Docusaurus"></a>
</h1>

<p align="center">
  <a href="https://www.npmjs.com/package/docusaurus"><a href="#backers" alt="sponsors on Open Collective"><img src="https://opencollective.com/Docusaurus/backers/badge.svg" /></a> <a href="#sponsors" alt="Sponsors on Open Collective"><img src="https://opencollective.com/Docusaurus/sponsors/badge.svg" /></a> <img src="https://img.shields.io/npm/v/docusaurus.svg?style=flat" alt="npm version"></a>
  <a href="https://circleci.com/gh/facebook/Docusaurus"><img src="https://circleci.com/gh/facebook/Docusaurus.svg?style=shield" alt="CircleCI Status"></a>
  <a href="CONTRIBUTING.md#pull-requests"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs Welcome"></a>
  <a href="https://discord.gg/docusaurus"><img src="https://img.shields.io/badge/chat-on%20discord-7289da.svg" alt="Chat"></a>
  <a href="https://github.com/prettier/prettier"><img alt="code style: prettier" src="https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square"></a>
  <a href="https://github.com/facebook/jest"><img src="https://img.shields.io/badge/tested_with-jest-99424f.svg" alt="Tested with Jest"></a>
</p>

## Introduction

A fork of the original [`docusaurus` package](https://www.npmjs.com/package/docusaurus).

This fork was created as a solution proposal for the issue [Support local search #776](https://github.com/facebook/Docusaurus/issues/776).

It's Not relly a **local** search solution since it's a server side search. But it allow the Docusaurus User to add your customized server search by implementing your own search engine an adding it as an Express Middleware.

## Installation

This Fork is available as [`@the-darc/docusaurus` package](https://www.npmjs.com/package/@the-darc/docusaurus) on [npm](https://www.npmjs.com).

```
npm i @the-darc/docusaurus --save-dev
npm i elasticlunr --save
```

## Usage

### siteConfig.js

Add the localSearch config in your `siteConfig.js` like this:

```js
  localSearch: {
    // Placeholder for the input-text
    placeholder: 'Local Search',
    // The express middleware that will resolver the end-point "/query?q=<search-string>"
    expressMiddleware: require('./local-search'),
    // Any customized configurations for your express middleware
    resultSnippet: {
      maxLength: 200,
      preLength: 28,
      posLength: 28
    }
  }
```

### Express Middleware Example

**File: <your-projeto>/local-search/index.js**
  
```
/**
 * Middleware express para processar uma requisição de pesquisa nos
 * documentos do Docusaurus.
 * 
 * Processa uma requisição que recebem como QueryParameter o atributo q com 
 * a string de pesquisa. E retorna como um JSON de resultados onde cada resultado
 * segue a interface:
 * {
 *   id: <String com ID do documento no indice de pesquisa>,
 *   score: <Number com o score do item para a pesquisa>,
 *   name: <Nome do tópico de documentação encontrado>,
 *   path: <Caminho do arquivo .md encontrado>,
 *   snippet: <trecho do texto com o termo pesquisado>
 * }
 */
var DocIndex = require('./DocIndex');

module.exports = searchMiddleware;

function searchMiddleware(siteConfig) {
    const docIndex = new DocIndex(siteConfig);

    return function searchMiddleware(req, res, next) {
        let q = req.query.q;
        if (!q) return res.status(404).send([]);
    
        var r = docIndex.search(q);
        res.status(200).send(r);
    };
}
```

**File: <your-projeto>/local-search/DocIndex.js**

```
const path = require('path');
const SIDEBARS = require('../sidebars.json');
const elasticlunr = require('elasticlunr');
const removeMd = require('remove-markdown');
const fs = require('fs');

function extractTopicTitle(content, defaultTitle) {
    let tmp = /title\:(.*)/ig.exec(content);
    return tmp ? tmp[1].trim() : defaultTitle;
}

function escapeRegExp(text) {
    return text.replace(/[-[\]{}()*+?.,\\/^$|#\s]/g, '\\$&');
}

class DocIndex {
    constructor(siteConfig) {
        this.resultSnippet = (siteConfig.localSearch || {}).resultSnippet || {};
        this.resultSnippet.maxLength = this.resultSnippet.maxLength || 200;
        this.resultSnippet.preLength = this.resultSnippet.preLength || 28;
        this.resultSnippet.posLength = this.resultSnippet.posLength || 28;

        this.docsPath = siteConfig.customDocsPath ? siteConfig.customDocsPath : 'docs';

        let documents = this._loadDocuments();
        this._index = this._createIndex(documents);

    }

    _loadDocuments() {
        var documents = [];
        Object.keys(SIDEBARS).forEach(function(helpKey) {
            var help = SIDEBARS[helpKey];
            Object.keys(help).forEach(function(sectionName) {
                var section = help[sectionName];
                section.forEach(function(topicPath) {
                    let fullPath = '../docs/' + topicPath + '.md';
                    try {
                        let content = fs.readFileSync(fullPath, 'utf8');
                        let title = extractTopicTitle(content);
                        let topicName = sectionName + ' > ' + title;
                        documents.push({
                            name: topicName,
                            topicPath: topicPath,
                            content: removeMd(content, {useImgAltText: false}).replace(/ \|/g, '')
                        });
                    } catch (e) {
                        if (e && e.code === 'ENOENT') {
                            console.warn('[WARN] File not found: "' + fullPath + '"');
                        } else {
                            console.error('[ERROR] Unexpected error reading file contents for indexing: "' + fullPath + '"');
                            throw e;
                        }
                    }
                });
            });
        });
        return documents;
    }

    _createIndex(documents) {
        var index = elasticlunr(function () {
            this.setRef('id');
            this.addField('content');
            // this.saveDocument(false);
        });
    
        documents.forEach(function (doc, id) {
            doc.id = id;
            index.addDoc(doc);
        });

        console.log('Index created. Used ' + Math.round(JSON.stringify(index).length/1024) + 'Kb');
        return index;
    }

    _parseResults(query, results) {
        var parsed = [];
        (results || []).forEach((result) => {
            var regex = new RegExp('.{0,'+this.resultSnippet.preLength+'}'+escapeRegExp(query)+'.{0,'+this.resultSnippet.posLength+'}', 'ig');
            var doc = this._index.documentStore.docs[result.ref];
            if (doc) {
                var snippets = [];
                var snippet = (regex.exec(doc.content) || [])[0];
                var totalLength = 0;
                while (snippet && totalLength < this.resultSnippet.maxLength) {
                    if (snippet.toUpperCase().indexOf(query.toUpperCase()) >= this.resultSnippet.preLength) {
                        // Remove a primeira palavra dado que ela potencialmente é uma palavra cortada
                        snippet = snippet.replace(/[^ ]*/, '');
                    }
                    if ( (snippet.length - snippet.toUpperCase().indexOf(query.toUpperCase())) >= this.resultSnippet.posLength) {
                        // Remove a última palavra dado que ela potencialmente é uma palavra cortada
                        snippet = snippet.replace(/[^ ]*$/, '');
                    }
                    totalLength += snippet.length;
                    snippets.push(snippet.trim());
                    snippet = (regex.exec(doc.content) || [])[0];
                }
                snippets = snippets.join(' (...) ');
                parsed.push({
                    id: result.ref,
                    score: result.score,
                    name: doc.name,
                    path: path.join(this.docsPath, doc.topicPath),
                    snippet: snippets
                });
            }
        });
        return parsed;
    }

    search(query) {
        var results = this._index.search(query, {});
        return this._parseResults(query, results);
    }

    saveIndex(fileUri) {
        fs.writeFile(fileUri || './index.json' , JSON.stringify(this._index), function (err) {
            if (err) throw err;
            console.log('index saved in index.json');
        });
    }
}

module.exports = DocIndex;

// --- test ---------------------------------------------------------
/*
var docIndex = new DocIndex({
    resultSnippet: {
        maxLength: 200,
        preLength: 28,
        posLength: 28
    }
});

console.log(JSON.stringify({
    result: docIndex.search('sydle')
}, null, 4));
*/
```


