# mod-data-loader

Copyright (C) 2017-2018 The Open Library Foundation

This software is distributed under the terms of the Apache License,
Version 2.0. See the file "[LICENSE](LICENSE)" for more information.

## Introduction

RMB-based module used to load test data.

Currently supports loading binary MARC records into the mod-inventory-storage instance table.

## APIs
Exposes six APIs
1. POST `/load/marc-rules` - uploads a [rules json](https://github.com/folio-org/test-data-loader/blob/master/ramls/rules.json) file to use when mapping marc fields to instance fields. The rules file is only stored in memory and will be associated with the tenant passed in the x-okapi-tenant header
2.  GET `/load/marc-rules`
3. POST `/load/marc-data?storageURL=http://localhost:8888` - posts the attached binary MARC file. This will convert the MARC records into instances and bulk load them into Postgres.
4. POST `/load/marc-data/test` - normalizes the attached binary MARC file (should be small) and returns json instance object as the response to the API. Can be used to check mappings from MARC to instances. The file attached should be kept small so that there are no memory issues for the client (up to 500 entries)
5. POST `/load/static?storageURL=http://localhost:8888` - posts static data (json template file) into the table specified in the template
6. POST `/load/static/test` - posts (but does not persist) static data, returning the actual records that would have been persisted in non-test mode

The RAML can be found here:
https://github.com/folio-org/data-loader/blob/master/ramls/loader.raml

### Some notes

 1. A tenant must be passed in the x-okapi-tenant header.
 2. A rules files must be set for that tenant.
 3. The inventory-storage module must be available at the host / port indicated via the storageURL query parameter (this is checked before processing begins). Direct access to mod-inventory-storage at storageURL is required to invoke /admin/importSQL, that endpoint is not available when invoked via Okapi.

### Example invocation

Run mod-inventory-storage in first console:

    cd mod-inventory-storage
    mvn clean install
    export DB_DATABASE=postgres
    export DB_HOST=localhost
    export DB_PASSWORD=postgres
    export DB_PORT=5433
    export DB_USERNAME=postgres
    java -jar target/mod-inventory-storage-fat.jar -Dhttp.port=8080

Run mod-data-loader using default port 8081 in second console:

    cd mod-data-loader
    mvn clean install
    java -jar target/mod-data-loader-fat.jar

Run data loader in third console:

    cd mod-data-loader
    curl -s -S -D - -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" \
      -X POST http://localhost:8080/_/tenant
    curl -s -S -D - -H "X-Okapi-Tenant: diku" -H "Content-type: application/octet-stream" -H "Accept: text/plain" \
      -d \@src/test/resources/rules.json \
      http://localhost:8081/load/marc-rules
    curl -s -S -D - -H "X-Okapi-Tenant: diku" -H "Content-type: application/octet-stream" -H "Accept: text/plain" \
      -d \@src/test/resources/msplit00000000.mrc \
      http://localhost:8081/load/marc-data?storageURL=http://localhost:8080\&storeSource=true

### MARC files

It is best to attach MARC files with the same amount of records as the batch size - this is not mandatory (default batch size is 50,000 records and can be changed via the `batchSize` query parameter)

[MarcEdit](http://marcedit.reeset.net/) can be used to split very large MARC records.

You can call the `/load/marc-data` API multiple times on different MARC files - this should improve loading performance (the amount of concurrent calls depends on the amount of hardware on the server)

A records position in the uploaded file will be present in the `X-Unprocessed` header for each MARC record that was not parsed correctly.

### Conversion rules

Control fields can be used to insert constant values into instance fields. For example, the below will insert the value Books into the instanceTypeId field if all conditions of this rule are met. Multiple rules may be declared. The `LDR` field indicates that the condition should be tested against the MARC's Leader field data.

```
 "rules": [
   {
     "conditions": [
       {
         "type": "char_select",
         "parameter": "0",
         "value": "7"
       },
       {
         "type": "char_select",
         "parameter": "1",
         "value": "8"
       },
       {
         "type": "char_select",
         "parameter": "0",
         "value": "0",
         "LDR": true
       }
     ],
     "value": "Books"
   }
 ]
```

#### Available functions

 - `char_select` - select a specific char (parameter) from the field and compare it to the indicated value (value) - ranges can be passed in as well (e.g. 1-3). `LDR` indicates that the data from the leader field should be used for this condition and not the data of the field itself
 - `remove_ending_punc` remove punctuation at the end of the data field (;:,/+=<space> as well as period handling .,..,...,....)
 - `trim_period` if the last char in the field is a period it is removed
 - `trim` remove leading and trailing spaces from the data field

Example:
```
 "rules": [
   {
     "conditions": [
       {
         "type": "remove_ending_punc,trim"
       }
     ]
   }
 ]
```
Note that you can indicate the use of multiple functions by using the comma delimiter. This is only possible for functions that do not receive parameters.
- `custom` - define a custom JavaScript function to run on the field's data (passed in as DATA to the JavaScript function as a bound variable. Must return a String value). For example:

```
"target": "publication.dateOfPublication",
"rules": [
  {
    "conditions": [
      {
        "type": "custom",
        "value": "DATA.replace(/\\D/g,'');"
      }
    ]
  }
]
```

#### Mapping partial data

To set an instance field with part of the data appearing in a specific subfield,
```
"rules": [
  {
    "conditions": [
      {
        "type": "char_select",
        "parameter": "35-37"
      }
    ]
  }
]
```
Notice that there is no `value` in the condition and no constant value set at the `rule` level.

#### Multiple subfields

Indicating multiple subfields will concatenate the values of each subfield into the target instance field:
```
"690": [
    {
      "subfield": ["a","y","5"],
      "description": "local subjects",
      "target": "subjects"
    }
  ]
```

#### Grouping fields into an object

Normally, all mappings in a single field that refer to the same object type will be mapped to a single object. For example, the below will map two subfields found in 001 to two different fields in the same identifier object within the instance.

```
  "001": [
    {
      "target": "identifiers.identifierTypeId",
      "description": "Type for Control Number (001)",
      ....
    },
    {
      "target": "identifiers.value",
      "description": "Control Number (001)",
  ...
    }
  ],
```
However, sometimes there is a need to create multiple objects (for example, multiple identifier objects) from subfields within a single field.
Consider the following MARC field:

`020    ##$a0877790019$qblack leather$z0877780116 :$c$14.00`

which should map to:
```
"identifiers": [
  { "value": "0877790019", "identifierTypeId": "8261054f-be78-422d-bd51-4ed9f33c3422"},
  { "value": "0877780116", "identifierTypeId": "8261054f-be78-422d-bd51-4ed9f33c3422"}
]
```

To achieve this, you can wrap multiple subfield definitions into an `entity` field. In the below, both subfields will be mapped to the same object (which would happen normally), however, any additional entries outside of the `"entity"` definitions will be mapped to a new object, hence allowing you to create multiple objects from a single MARC field.
```
"020": [
    {
      "entity": [
        {
        "target": "identifiers.identifierTypeId",
        "description": "Type for Control Number (020)",
        "subfield": ["a"],
        ...
        },
        {
          "target": "identifiers.value",
          "description": "Control Number (020)",
          "subfield": ["b"],
          ...
        }
      ]
    },
```

##### Handling repeating fields

The `entity` example will concatenate together values from repeated fields. For example, an `entity` on subfield  "a" will concatenate all values in all the "a" subfields (if they repeat) - and map the concatenated value to the declared field. If there is a need to have each "a" subfield generate its own object within the instance (for example, each "a" subfield should create a separate classification entry and should not be concatenated within a single entry). The following field can be added to the configuration: `"entityPerRepeatedSubfield": true`

```
 "050": [
    {
      "entityPerRepeatedSubfield": true,
      "entity": [
        {
          "target": "classifications.classificationTypeId",
          "subfield": ["a"],
          "rules": [
            {
              "conditions": [],
              "value": "99999999-be78-422d-bd51-4ed9f33c3422"
            }
          ]
        },
        {
          "target": "classifications.classificationNumber",
          "subfield": ["a"]
        }
      ]
    },
    {
      "entityPerRepeatedSubfield": true,
      "entity": [
        {
          "target": "classifications.classificationTypeId",
          "subfield": ["b"],
          "rules": [
            {
              "conditions": [],
              "value": "99999999-be78-422d-bd51-4ed9f33c3423"
            }
          ]
        },
        {
          "target": "classifications.classificationNumber",
          "subfield": ["b"]
        }
      ]
    }
  ],

```

#### Delimiting subfields

As previously mentioned, grouping subfields  `"subfield": [ "a", "y", "5" ]` will concatenate (space delimited) the values in those subfields and place the result in the target. However, if there is a need to declare different delimiters per set of subfields, the following can be declared using the `"subFieldDelimiter"` array:

```
  "600": [
    {
      "subfield": [
        "a","b","c","d","v","x","y","z"
      ],
      "description": "",
      "subFieldDelimiter": [
        {
          "value": "--",
          "subfields": [
            "d","v","x","y","z"
          ]
        },
        {
          "value": " ",
          "subfields": ["a", "b", "c"]
        },
        {
          "value": "&&&",
          "subfields": []
        }
      ],
      "target": "subjects"
    }
  ]
```
An empty subfields array indicates that this will be used to separate values from different subfield sets (subfields associated with a specific separator).

#### A single subfield into multiple subfields

It is sometimes necessary to parse data in a single subfield and map the output into multiple subfields before processing.
For example:
`041 $aitaspa`
We may want to take this language field and convert it into two $a subfields before we begin processing. This can be achieved in the following manner:

```
"041": [
  {
    "entityPerRepeatedSubfield": true,
    "entity": [
      {
        "subfield": ["a"],
        "subFieldSplit": {
          "type": "custom",
          "value": "DATA.match(/.{1,3}/g)"
        },
        "rules": [
          {
            "conditions": [
              {
                "type": "trim"
              }
            ]
          }
        ],
        "description": "",
        "target": "languages"
      }
...
```

Once pre-processing is complete, the regular rules / mappings will be applied - this includes the entity option which can map each of the newly created subfields into separate objects.

There are currently 2 functions that can be called to parse the data within a single subfield:

 **`split_every`** which receives a value indicating a hard split every n characters
```
"subFieldSplit": {
   "type": "split_every",
   "value": "3"
 },
```

**`custom`** - which receives a JavasSript function and must return a string array representing the new values generated from the original data.
```
"subFieldSplit": {
  "type": "custom",
  "value": "DATA.match(/.{1,3}/g)"
}
```

#### Processing rules on concatenated data

By default rules run on the data in a single subfield, hence, concatenated subfields concatenate normalized data. In order to concatenate
un-normalized data, and run the rules on the concatenated data add the following field: `"applyRulesOnConcatedData": true,`
This can be used when punctuation should only be removed from the end of a concatenated string.
```
"500": [
    {
      "subfield": [
        "a"
      ],
      "applyRulesOnConcatenatedData": true,
      "description": "",
```

#### JSON fields supported only on data field (not control fields)

1. `subFieldSplit`
2. `subFieldDelimiter`
3. `applyRulesOnConcatenatedData`
4. `entity`
5. `entityPerRepeatedSubfield`


**Note**:

Currently, if the database is down, or the tenant in the x-okapi-tenant does not exist, the API will return success but will do nothing. This is an issue in the RMB framework used by mod-inventory-storage (errors will be logged in the mod-inventory-storage log, but the message is not propagated at this time).


**Performance**

A single call to the API with a binary MARC file with 50,000 records should take approximately 40 seconds. You can run multiple API calls concurrently with different files to speed up loading. A 4-core server should support at least 4 concurrent calls (approximately 200,000 records within a minute).

Adding JavaScript custom functions (while allowing maximum normalization flexibility) does slow down processing. Each call takes approximately 0.2 milliseconds, meaning, for example, attaching custom JavaScript functions to 6 fields in a 50,000 record MARC file means 300,000 JavaScript calls at 0.2 milliseconds per call -> 60,000 milliseconds (60 seconds) overhead.


### Static data

The loader allows loading static data into a tenant's table by calling the `/load/static` API.

For example, calling the `/load/static` API with the following attachment:
```
{
  "record": {
    "id": "${randomUUID}"
  },
  "values": {
    "name": ["Book", "Journal" , "Video" , "Audio" , "Map"]
  },
  "type": "material_type"
}
```
That will create five records and attempt to persist them into the `type` table:
```
{"id":"33f2d57a-3a56-45f2-8a06-1e53d7c558a3","name":"Book"}
{"id":"af195d6d-da0e-4b6f-a0a0-ebd5163ecdba","name":"Journal"}
{"id":"40fba3d9-68b2-4b97-9608-f72799ac00df","name":"Video"}
{"id":"fb630301-44a0-4a36-ad3d-6f4c86e2a7b8","name":"Audio"}
{"id":"58f1e02e-35a5-43cb-af4c-129478b3a3a9","name":"Map"}
```

 - The `${randomUUID}` indicates that the "id" field should be populated with a server generated UUID
 - The "values" field indicates which field name / values to use when generating the records

Example 2: (define an array with static ids)

```json
{
  "record": {
    "a": "val1",
    "b": "val2"
  },
  "values":
    [
      {"id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfd","name":"Book"},
      {"id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfc","name":"Journal"},
      {"id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfb","name":"Video"}
    ]
  ,
  "type": "material_type"
}

That will create three records:
```
{"a":"val1","b":"val2","id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfd","name":"Book"}
{"a":"val1","b":"val2","id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfc","name":"Journal"}
{"a":"val1","b":"val2","id":"9d5f9eb6-b92e-4a1a-b4f5-310bc38dacfb","name":"Video"}
```

*Note* that the only validation occurring is the validation done on the DB layer (duplicate keys, etc.).

## Additional information

### Other documentation

Other [modules](https://dev.folio.org/source-code/#server-side) are described,
with further FOLIO Developer documentation at [dev.folio.org](https://dev.folio.org/)

### Issue tracker

See project [DIMPT](https://issues.folio.org/browse/DIMPT)
at the [FOLIO issue tracker](https://dev.folio.org/guidelines/issue-tracker).

## Documentation for `Processor.java`

See references within the code. Larger explanations have been moved here for code cleaning

### constant Value

get the constant value (if is was declared) to set the instance field to in case all
conditions are met for a rule, since there can be multiple rules
each with multiple conditions, a match of all conditions in a single rule
will set the instance's field to the const value. hence, it is an AND
between all conditions `pr.doBreak()` ?and an OR between all rules
example of a constant value declaration in a rule:
```
"rules": [
           {
             "conditions": [.....],
             "value": "book"
```
if this value is not indicated, the value mapped to the instance field will be the
output of the function - see below for more on that

### Functions

1..n functions can be declared within a condition (comma delimited).
for example:
A condition with with one function, a parameter that will be passed to the
function, and the expected value for this condition to be met
```json
{
  "type": "char_select",
  "parameter": "0",
  "value": "7"
}
```
the functions here can rely on the single value field for comparison
to the output of all functions on the marc's field data
or, if a custom function is declared, the value will contain
the javascript of the custom function
for example:
```
"type": "custom",
"value": "DATA.replace(',' , ' ');"
```

### Delimiters

allow to declare a delimiter when concatenating subfields.
also allow , in a multi subfield field, to have some subfields with delimiter x and
some with delimiter y, and include a separator to separate each set of subfields
maintain a delimiter per subfield map - to lookup the correct delimiter and place it in string
maintain a string buffer per subfield - but with the string buffer being a reference to the
same stringbuilder for subfields with the same delimiter - the stringbuilder references are
maintained in the buffers2concat list which is then iterated over and we place a separator
between the content of each string buffer reference's content

