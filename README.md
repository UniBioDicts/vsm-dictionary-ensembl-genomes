# vsm-dictionary-ensembl-genomes

<!-- badges: start -->
[![Node.js CI](https://github.com/UniBioDicts/vsm-dictionary-ensembl-genomes/workflows/Node.js%20CI/badge.svg)](https://github.com/UniBioDicts/vsm-dictionary-ensembl-genomes/actions)
[![codecov](https://codecov.io/gh/UniBioDicts/vsm-dictionary-ensembl-genomes/branch/master/graph/badge.svg)](https://codecov.io/gh/UniBioDicts/vsm-dictionary-ensembl-genomes)
[![npm version](https://img.shields.io/npm/v/vsm-dictionary-ensembl-genomes)](https://www.npmjs.com/package/vsm-dictionary-ensembl-genomes)
[![Downloads](https://img.shields.io/npm/dm/vsm-dictionary-ensembl-genomes)](https://www.npmjs.com/package/vsm-dictionary-ensembl-genomes)
[![License](https://img.shields.io/npm/l/vsm-dictionary-ensembl-genomes)](#license)
<!-- badges: end -->

## Summary

`vsm-dictionary-ensembl-genomes` is an implementation 
of the 'VsmDictionary' parent-class/interface (from the package
[`vsm-dictionary`](https://github.com/vsm/vsm-dictionary)), that uses 
the [EBI Search RESTful Web Services](https://www.ebi.ac.uk/ebisearch/apidoc.ebi) 
to interact with the Ensembl genome database for non-vertebrate species and 
translate the provided gene information into a VSM-specific format.

## Install

Run: `npm install`

## Example use

### Node.js

Create a directory `test-dir` and inside run `npm install vsm-dictionary-ensembl-gemones`.
Then, create a `test.js` file and include this code for example:

```javascript
const DictionaryEnsemblGenomes = require('vsm-dictionary-ensembl-genomes');
const dict = new DictionaryEnsemblGenomes({log: true});

dict.getEntryMatchesForString('tp53', { page: 1, perPage: 10 }, 
  (err, res) => {
    if (err) 
      console.log(JSON.stringify(err, null, 4));
    else
      console.log(JSON.stringify(res, null, 4));
  }
);
```
Then, run `node test.js`

### Browsers

```html
<script src="https://unpkg.com/vsm-dictionary-ensembl-genomes@^1.0.0/dist/vsm-dictionary-ensembl-genomes.min.js"></script>
```
after which it is accessible as the global variable `VsmDictionaryEnsemblGenomes`.

## Tests

Run `npm test`, which runs the source code tests with Mocha.  
If you want to quickly live test the EBI Search API, go to the 
`test` directory and run:
```
node getEntries.test.js
node getEntryMatchesForString.test.js
```

## 'Build' configuration

To use a VsmDictionary in Node.js, one can simply run `npm install` and then
use `require()`. But it is also convenient to have a version of the code that
can just be loaded via a &lt;script&gt;-tag in the browser.

Therefore, we included `webpack.config.js`, which is a Webpack configuration file for 
generating such a browser-ready package.

By running `npm build`, the built file will appear in a 'dist' subfolder. 
You can use it by including: 
`<script src="../dist/vsm-dictionary-ensembl-genomes.min.js"></script>` in the
header of an HTML file. 

## Specification

Like all VsmDictionary subclass implementations, this package follows
the parent class
[specification](https://github.com/vsm/vsm-dictionary/blob/master/Dictionary.spec.md).
In the next sections we will explain the mapping between the data 
offered by EBI Search's API and the corresponding VSM objects. Find the 
documentation for the API here: https://www.ebi.ac.uk/ebisearch/documentation.ebi

Note that if we receive an error response from the EBI Search servers (see the 
URL requests for `getEnties` and `getEntryMatchesForString` below) that is not a
JSON string that we can parse, we formulate the error as a JSON object ourselves 
in the following format:
```
{
  status: <number>,
  error: <response> 
}
```
where the *response* from the server is JSON stringified.

### Map Ensembl Genomes to DictInfo VSM object

This specification relates to the function:  
 `getDictInfos(options, cb)`

If the `options.filter.id` is not properly defined 
or the `http://www.ensemblgenomes.org` dictID is included in the 
list of ids used for filtering, `getDictInfos` returns a static object 
with the following properties:
- `id`: 'http://www.ensemblgenomes.org' (will be used as a `dictID`)
- `abbrev`: 'Ensembl Genomes'
- `name`: 'Ensembl Genomes'

Otherwise, an empty result is returned.

### Map Ensembl Genomes to Entry VSM object

This specification relates to the function:  
 `getEntries(options, cb)`

Firstly, if the `options.filter.dictID` is properly defined and in the list of 
dictIDs the `http://www.ensemblgenomes.org` dictID is not included, then 
an **empty array** of entry objects is returned.

If the `options.filter.id` is properly defined (with IDs like
`http://www.ensemblgenomes.org/id/AT3G52430`) then we use a query like this:

```
https://www.ebi.ac.uk/ebisearch/ws/rest/ensemblGenomes_gene/entry/Z208_01625,EMPG_14124,AT3G52430?fields=id%2Cname%2Cdescription%2Cgene_synonyms%2Cspecies&format=json
```

For the above URL, we provide a brief description for each sub-part: 
- The first part refers to the EBI Search's main REST endpoint: https://www.ebi.ac.uk/ebisearch/ws/rest/
- The second part refers to the **domain** of search (*ensemblGenomes_gene*)
- The third part refers to the *entry* endpoint (which allows us to request 
for entry information associated with entry identifiers)
- The fourth part is the *entry IDs*, comma separated (we extract the last part 
of the EnsemblGenomes-specific URI for each ID). Note that for VSM the URI ID is 
something like: `http://www.ensemblgenomes.org/id/Z208_01625`, 
while the EnsemblGenomes entry IDs are created based on the IDs imported from 
external sources (for more info see [here](http://ensemblgenomes.org/info/data/identifiers)).
- The fifth part is the *fields* of interest - i.e. the information related to 
the entries that we will map to VSM-entry properties. For a complete list of the 
available fields for the ensemblGenomes_gene domain, see: https://www.ebi.ac.uk/ebisearch/metadata.ebi?db=ensemblGenomes_gene
- The last part defines the format of the returned data (JSON)

Otherwise, we ask for all ids (by default **id sorted**) with this query:
```
https://www.ebi.ac.uk/ebisearch/ws/rest/ensemblGenomes_gene?query=domain_source:ensemblGenomes_gene&fields=id%2Cname%2Cdescription%2Cgene_synonyms%2Cspecies&sort=id&size=50&start=0&format=json
```

Note that depending on the `options.page` and `options.perPage` options 
we adjust the `size` and `start` parameters accordingly. The `size` requested 
can be between 0 and 100 and if its not in those limits or not properly defined, 
we set it to the default page size which is **50**. The `start` (offset, zero-based)
can be between 0 and 1000000. The default value for `start` is 0 (if `options.page`
is not properly defined) and if the *(page size) \* (#page requested - 1)* exceeds 1000000, then we set it to `999999`, 
allowing thus the retrieval of the last entry (EBI Search does not allow us to 
retrieve more than the 1000000th entry of a domain).

Only when requesting for specific IDs, we sort the results depending on the
`options.sort` value: results can be either `id`-sorted or `str`-sorted,
according to the specification of the parent 'VsmDictionary' class.
We then prune these results according to the values `options.page` (default: 1)
and `options.perPage` (default: 50).

When using the EBI search API, we get back a JSON object with an *entries* 
property, which has as a value an array of objects (the entries). Every entry
object has a *fields* property whose value is an object with properties all 
the fields that we defined in the initial query. We now provide a mapping of 
these fields to VSM-entry specific properties:

EnsemblGenomes field | Type | Required | VSM entry/match object property | Notes  
:---:|:---:|:---:|:---:|:---:
`id` | Array | YES | `id`,`str`,`terms[0].str` | The VSM entry id is the full URI. We use the simple id string for `str` when `name` is empty.
`name` | Array | NO | `str`,`terms[0].str` | We use the first element only.
`description` | Array | NO | `descr` | We use the first element only
`gene_synonyms` | Array | NO | `terms[i].str` | We map the whole array
`species` | Array | NO | `z.species` | We use the first element only

Note that the above mapping describes what we as developers thought as the most
reasonable. There is though a global option `optimap` that you can pass to the 
`DictionaryEnsemblGenomes` object, which optimizes the above mapping for curator clarity
and use. The **default value is true** and what changes in the mapping table
above (which is the mapping for `optimap: false` actually) is that the VSM's `descr` 
entry/match object property takes the combined value of the `species` (first two
words), the gene identifier and synonyms (`id`, `gene_synonym`) and the 
`description` (in that order). The reason behind this is that the `description` 
is sometimes the same for different genes or even empty when `optimap: false` 
and thus not distinguishable, so we had to provide a more clarified description 
string for each entry.

### Map Ensembl Genomes to Match VSM object

This specification relates to the function:  
 `getEntryMatchesForString(str, options, cb)`

Firstly, if the `options.filter.dictID` is properly defined and in the list of 
dictIDs the `http://www.ensemblgenomes.org` dictID is not included, then 
an **empty array** of match objects is returned.

Otherwise, an example of a URL string that is being built and send to the EBI 
Search's REST API when requesting for `tp53`, is:
```
https://www.ebi.ac.uk/ebisearch/ws/rest/ensemblGenomes_gene?query=tp53&fields=id%2Cname%2Cdescription%2Cgene_synonyms%2Cspecies&size=20&start=0&format=json
```

The fields requested are the same as in the `getEntries(options, cb)` 
case as well as the mapping shown in the table above. Also for the `size` and 
`start` parameters the same things apply as in the `getEntries` specification.
 
No sorting whatsoever is done on the server or client side.

## License

This project is licensed under the AGPL license - see [LICENSE.md](LICENSE.md).
