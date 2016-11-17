# shex.js
shex.js javascript implementation of Shape Expressions

This is cloned from the the TEisSE branch of [shexSpec/shex.js](https://github.com/shexSpec/shex.js/commit/f9b37f93f46b324b50661c235e0f1921d3bb15cc). Its purpose
is to be a static image of the shex.js validation tools at the point that the paper, tentatively titled 
"Modeling and Validating HL7 FHIR Profiles Using Semantic Web Shape Expressions (ShEx)" was written (version 1.5 as of November 17, 2016).

## Notes
The pre-commit hook doesn't currently pass.  It depends on a semi-independent repository [shexTest](https://github.com/shexSpec/shexTest) which doesn't appear to be fully synced
with this code.  If you need to update this repository, use the `-n` git commit option.

## install

```bash
> git clone https://github.com/caCDE-QA/shex.js
> cd shex.js
> npm install --save
```


## validation

You can validate RDF data using the executable `bin/validate` 

## REST Service

Starting a REST server:

```
./bin/validate -S http://localhost:4290/validate

Visit http://localhost:4290/ in browser.

Test with a supplied schema and data:
  curl -i http://localhost:4290/validate -F "schema=@test/cli/1dotOr2dot.shex" -F "shape=http://a.example/S1" -F "data=@test/cli/p2p3.ttl" -F "node=x"
or preload the schema and just supply the data:
  bin/validate -x test/cli/1dotOr2dot.shex -s http://a.example/S1 -S  http://localhost:4290/validate
and pass only the data parameters:
  curl -i http://localhost:4290/validate -F "data=@test/cli/p2p3.ttl" -F "node=x"
Note that shape and node can be relative or prefixed URLs.

Press CTRL+C to stop...
```

The following command is what is needed to use shex.js as a FHIR ShEx validations service:

```
./bin/validate -S http://localhost:4290/validate --regex-module nfax-val-1err --duplicate-shape ignore
```

The `regex-module` argument addresses an issue where the server takes an inordinate amount of time to validate certain FHIR types and the
`duplicate-shape` argument addresses a bug in the STU3 ShExGenerator where definitions for the starting resource are emitted more than once.

###  validation executable

Validate something in HTTP-land:

```
./bin/validate \
    -x http://shex.io/examples/Issue.shex \
    -d http://shex.io/examples/Issue1.ttl \
    -s http://shex.io/examples/IssueShape \
    -n http://shex.io/examples/Issue1
```

That validates node `http://shex.io/examples/Issue` in `http://shex.io/examples/Issue1.ttl` against shape `http://shex.io/examples/IssueShape` in `http://shex.io/examples/Issue.shex`.
The result is a JSON structure which tells you exactly how the data matched the schema.

```
{
  "type": "test",
  "node": "http://shex.io/examples/Issue1",
  "shape": "http://shex.io/examples/IssueShape",
  "solution": {
...
  }
}
```

Had we gotten a `null`, we'd know that the document was invalid with respect to the schema.
See the [ShExJ primer](http://shex.io/primer/) for a description of ShEx validation and the [ShExJ specification](http://shex.io/primer/ShExJ) for more details about the results format.

####  relative resolution

`validate`'s -n and -s arguemtns are evaluated as IRIs relative to the (first) data and schema sources respectively.
The above invocation validates the node `<Issue1>` in `http://shex.io/examples/Issue1.ttl`.
This and the shape can be written as relative IRIs:

```
./bin/validate \
    -x http://shex.io/examples/Issue.shex \
    -d http://shex.io/examples/Issue1.ttl \
    -s IssueShape \
    -n Issue1
```


### validation library

Parsing from the old interwebs involves a painful mix of asynchronous callbacks for getting the schema and the data and parsing the data (shorter path below):

```
var shexc = "http://shex.io/examples/Issue.shex";
var shape = "http://shex.io/examples/IssueShape";
var data = "http://shex.io/examples/Issue1.ttl";
var node = "http://shex.io/examples/Issue1";

var http = require("http");
var shex = require("shex");
var n3 = require('n3');

// generic async GET function.
function GET (url, then) {
  http.request(url, function (resp) {
    var body = "";
    resp.on('data', function (chunk) { body += chunk; });
    resp.on("end", function () { then(body); });
  }).end();
}

var Schema = null; // will be loaded and compiled asynchronously
var Triples = null; // will be loaded and parsed asynchronously
function validateWhenEverythingsLoaded () {
  if (Schema !== null && Triples !== null) {
    console.log(shex.Validator(Schema).validate(Triples, node, shape));
  }
}

// loaded the schema
GET(shexc, function (b) {
  // callback parses the schema and tries to validate.
  Schema = shex.Parser(shexc).parse(b)
  validateWhenEverythingsLoaded();
});

// load the data
GET(data, function (b) {
  // callback parses the triples and tries to validate.
  var db = n3.Store();
  n3.Parser({documentIRI: data}).parse(b, function (error, triple, prefixes) {
    if (error) {
      throw Error("error parsing " + data + ": " + error);
    } else if (triple) {
      db.addTriple(triple)
    } else {
      Triples = db;
      validateWhenEverythingsLoaded();
    }
  });
});
```

See? That's all there was too it!

OK, that's miserable. Let's use the ShExLoader to wrap all that callback misery:

```
var shexc = "http://shex.io/examples/Issue.shex";
var shape = "http://shex.io/examples/IssueShape";
var data = "http://shex.io/examples/Issue1.ttl";
var node = "http://shex.io/examples/Issue1";

var shex = require("shex");
shex.Loader([shexc], [], [data]).then(function (loaded) {
    console.log(shex.Validator(loaded.schema).validate(loaded.data, node, shape));
});
```

Note that the shex loader takes an array of ShExC schemas, either file paths or URLs, and an array of JSON schemas (empty in this invocation) and an array of Turtle files.

## conversion

As with validation (above), you can convert by either executable or library.

###  conversion executable

ShEx can be represented in the compact syntax
```
PREFIX ex: <http://ex.example/#>
<IssueShape> {                       # An <IssueShape> has:
    ex:state (ex:unassigned            # state which is
              ex:assigned),            #   unassigned or assigned.
    ex:reportedBy @<UserShape>        # reported by a <UserShape>.
}
```
or in JSON:
```
{ "type": "schema", "start": "http://shex.io/examples/IssueShape",
  "shapes": {
    "http://shex.io/examples/IssueShape": { "type": "shape",
      "expression": { "type": "eachOf",
        "expressions": [
          { "type": "tripleConstraint", "predicate": "http://ex.example/#state",
            "valueExpr": { "type": "valueClass", "values": [
                "http://ex.example/#unassigned", "http://ex.example/#assigned"
          ] } },
          { "type": "tripleConstraint", "predicate": "http://ex.example/#reportedBy",
            "valueExpr": { "type": "valueClass", "reference": "http://shex.io/examples/UserShape" }
          }
] } } } }
```

You can convert between them with shex-to-json:
```
./bin/shex-to-json http://shex.io/examples/Issue.shex
```
and, less elegantly, back with json-to-shex.

### conversion by library

As with validation, the ShExLoader wrapes callbacks and simplifies parsing the libraries:

```
var shexc = "http://shex.io/examples/Issue.shex";

var shex = require("shex");
shex.Loader([shexc], [], []).then(function (loaded) {
    console.log(JSON.stringify(loaded.schema, null, "  "));
});
```

There's no actual conversion; the JSON representation is just the stringification of the parsed schema.


## local files

Command line arguments which don't start with http:// or https:// are assumed to be file paths.
We can create a local JSON version of the Issues schema:
```
./bin/shex-to-json http://shex.io/examples/Issue.shex > Issue.json
```
and use it to validate the Issue1.ttl as we did above:
```
./bin/validate \
    -j Issue.json \
    -d http://shex.io/examples/Issue1.ttl \
    -s http://shex.io/examples/IssueShape \
    -n http://shex.io/examples/Issue1
```

Of course the data file can be local as well.

Happy validating!
