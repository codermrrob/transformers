# JSON Transformation Language

JTL is a language for transforming JSON into JSON. It is somewhat of a declarative language in so much as we describe what we want to have happen rather than the steps to create our target json (think like Excel).

It is built from the ground up to be able to transform business objects with relationships to other business objects that are described in json, not just transform the text. In other words we can create new objects with new relationships as well as transform property keys and values. To this end we use a combination of querying, copying, and creating json data as well as using string and other data manipulation to transform json structure, relationships, properties and data.

This is the beta version of the language, and is therefore a bit of a moving target but for the most part changes are not breaking and new features are introduced side by side whenever that is possible. Older features and functionality will start to be deprecated as they are replace. The beta status also means there are a few 'foibles' to know about, but these are documented and usually related to a short coming in the interpreter.

Enough chat, straight to an example.

## Start at the beginning

Lets start with a simple person object in json and just change a property name.

```json
{
"person" : {
        "firstName": "John",
        "lastName": "Smith",
        "age": 25,
        "address":
        {
            "streetAddress": "21 2nd Street",
            "city": "New York",
            "state": "NY",
            "postalCode": "10021"
        }
    }
}
```

We can rename `person` to `customer` by querying the source document (input) for the `person` object and placing the result of the query into the `customer` object in our target document (output).

The query is a JSON Path query and `$` just means the root of the document. JTL leans heavily on jsonPath, so it is recommended to get familiar with querying json using it. Where there are short comings in jsonPath, such as the inability to refer to a parent or keys JTL provides functions to help out.

```json
{
    "source" : {
        "path" : "$.person",
        "type" : "Object"
    },
    "target" : {
        "path" : "customer",
        "type" : "Object"
    }
}
```

Try this out using the JTL runner service at <https://<coming-soon!>/api/jtl/run>. You need to provide the input and the jtl in the body of the request like this:

```json
{
   "input":{
      "person":{
         "firstName":"John",
         "lastName":"Smith",
         "age":25,
         "address":{
            "streetAddress":"21 2nd Street",
            "city":"New York",
            "state":"NY",
            "postalCode":"10021"
         }
      }
   },
   "transformer":{
      "mappingSpecificationRoot":{
         "mappingSpecification":{
            "propertyMaps":[
               {
                  "source":{
                     "path":"$.person",
                     "type":"Object"
                  },
                  "target":{
                     "path":"customer",
                     "type":"Object"
                  }
               }
            ]
         }
      }
   }
}
```

A little bit of scaffold is unavoidable. The mappingSpecificationRoot object is a bit superfluous here but it is required because a  transformation can have more than one part including both c# and python script.

A `mappingSpecification` is a core concept in JTL. It literally specifies the output in terms of structure and queries and data transformations of the input.

At it's simplest a `mappingSpecification` is just a collection of simple property to property maps.

You can run this simple jtl sample:

```http
https://<coming-soon!>/api/jtl/run
content-type: application/json

/*the above json for the request body*/

```

You will receive the response which contains json:

```json
{
  "output": {
    "customer": {
      "firstName": "John",
      "lastName": "Smith",
      "age": 25,
      "address": {
        "streetAddress": "21 2nd Street",
        "city": "New York",
        "state": "NY",
        "postalCode": "10021"
      }
    }
  },
  "messages": [],
  "metrics": {
    "mTicks": 200,
    "mops": 1,
    "scriptTicks": 0
  }
}
```

The `messages` property is used to report any warnings or errors. The `metrics` object shows you the execution time in ticks (10,000 ticks in a millisecond or 10,000,000 ticks per second if you prefer), `mops` is the count of mapping operations executed.

## Create An Object

Before we copied an object from the input, renamed it and placed it in the output. This time we are creating an object from nothing. Also we introduce functions, which all start with a `!`.

```json
{
    "source" : {
        "path" : "!new('Object')",
        "type" : "Object"
    },
    "target" : {
        "path" : "Address",
        "type" : "Object"
    }
}
```

So to make that slightly more interesting, using the `person` object again:

```json
{
   "input":{
      "person":{
         "firstName":"John",
         "lastName":"Smith",
         "age":25,
         "address":{
            "streetAddress":"21 2nd Street",
            "city":"New York",
            "state":"NY",
            "postalCode":"10021"
         }
      }
   },
   "transformer":{
      "mappingSpecificationRoot":{
         "mappingSpecification":{
            "propertyMaps":[
               {
                  "source":{
                     "path":"$.person",
                     "type":"Object"
                  },
                  "target":{
                     "path":"customer",
                     "type":"Object"
                  }
               },
               {
                  "source":{
                     "path":"!new('Object')",
                     "type":"Object"
                  },
                  "target":{
                     "path":"newAddressObj",
                     "type":"Object"
                  },
                  "mappingSpecification":{
                     "propertyMaps":[
                        {
                           "source":{
                              "path":"$.person.address.streetAddress",
                              "type":"String"
                           },
                           "target":{
                              "path":"address1",
                              "type":"String"
                           }
                        },
                        {
                           "source":{
                              "path":"!concat($.person.address.city, ', ', $.person.address.state, ', ' , $.person.address.postalCode)",
                              "type":"String"
                           },
                           "target":{
                              "path":"address2",
                              "type":"String"
                           }
                        }
                     ]
                  },
               },
               {
                "source":{
                    "path":"",
                    "type":"String"
                },
                "target":{
                    "path":"",
                    "type":"String"
                }
               }
            ]
         }
      }
   }
}
```

## Mapping Variables

Variables are not really variables, they are named constants because they are immutable. They also suffer from a lack of sophistication at the present time and function only as a substitution in expressions.

But they still server a valuable purpose enabling configuration options and more.

A mapping specification can have an array of mapping variables.
We can use them to do arithmetic. Which is the only way possible in the current version as query paths are not correctly evaluated first.

```json
"mappingSpecification":{
    "propertyMaps":[
        {
            "source":{
                "path":"$.person",
                "type":"Object"
            },
            "target":{
                "path":"customer",
                "type":"Object"
            },
            "mappingSpecification" : {
                "mappingVariables" : [
                    {
                        "name" : "@height",
                        "path" : "$.person.height",
                        "type" : "Decimal"
                    },
                    {
                        "name" : "@weight",
                        "path" : "$.person.weight",
                        "type" : "Decimal"
                    }
                ],
                "propertyMaps" : [
                    {
                        "source":{
                            "path":"@weight / Pow(@height,2)",
                            "type":"Decimal"
                        },
                        "target":{
                            "path":"bmi",
                            "type":"Decimal"
                        }
                    }
                ]
            }
        }
    ]
}

```

Arithmetic is currently done using NCalc so all operators & functions available with NCalc are available.

## A Little about Types

JTL has the following types defined:

* String
* Int
* Long
* Datetime
* Decimal
* Boolean
* Collection
* Object
* Null
* JtlString
* IObject
* IArray

Most are self explanatory. When the source type and the target type are different the JTL will attempt to convert from the source type to the target type, using the culture specified in the integration stage.

### `String` vs `JtlString`

`String` is for all normal uses of a string. Note that in JTL (like all languages) string constants are surrounded with either single quotes or double quotes. Don't get confused that the json property/value pair that the JTL statement is stored in also has quotes around it. The json quotes are invisible to the JTL because they are there for the json parser. The JTL sees the property values *after* the json has been parsed. This means that a string constant will always require its own quotes.

(I like to prefix my variables with a $ character, this is purely personal and while I recommend prefixing variables you can use any character - and another might be better, to avoid confusion with $. jsonpah syntax)

{
   "name" : "$myString",
   "path" : "'some string value'",
   "type" : "String"
}

`JtlString` is a string variation which has one useful application. Dynamically building queries from configuration or source data. See more on Dynamic Queries below.

### `IObject` and `ICollection`

When mapping a collection or object The normal JTL execution will take a copy of the source data automatically apply any `mappingSpecification` and place the result in the target. For most of the time this is fine, but there are functions that also use a mapping specification (e.g. !copyObjectWithMapping). With these functions it is rare that we want the default execution flow because the mapping would be applied twice - once by the function and once by the default execution flow that applies the mapping to the result of the source path expression. The IObject & ICollection types prevent this from happening. The 'I' means immutable. In other words, any provided mapping specification, for this object or collection will be ignored by the default execution flow. Setting the *target* type as an Immutable `IObject` will prevent the automatic mapping specification execution when setting the target.

## Inline declared functions

There is a fairly unsophisticated method of creating your own inline functions.

`!asFunc( !let( '@height', $.height, 'Decimal' ) , !let( '@weight', $.weight, 'Decimal' ), @weight / Pow(@height,2) )`

The two variables are created and available to be applied in the calculation.
The result of one function is made available to the next. This will be revisited later in the docs with grouping examples.

## Arrays

Arrays can be processed in two ways, using the functions `!mapEachAdd` and `!mapEachMerge`.
`!mapEachAdd` will add objects to an array, whereas merge will merge objects or properties into an array.

For example this mapping

```json
{
   "mapId" : "10",
   "source":{
      "path":"!mapEachAdd( $.people[*] )",
      "type":"Collection"
   },
   "target":{
      "path":"customers",
      "type":"Collection"
   },
   "mappingSpecification" : {
      "propertyMaps" : [
            {
               "source":{
                  "path":"!concat($.firstName, ' ', $.lastName)",
                  "type":"String"
               },
               "target":{
                  "path":"name",
                  "type":"String"
               }
            }
      ]
   }
}
```

will produce

```json
"customers": [
   "John Smith",
   "Simon Pieman"
]
```

But if you change the map to a `!mapEachAdd` you'll produce an array of simple objects:

```json
"customers": [
   {
      "name": "John Smith"
   },
   {
      "name": "Simon Pieman"
   }
]
```

Getting slightly more sophisticated with arrays we can create named objects also:

```json
{
   "mapId":"10",
   "source":{
      "path":"!mapEachAdd( $.people[*] )",
      "type":"Collection"
   },
   "target":{
      "path":"customers",
      "type":"Collection"
   },
   "mappingSpecification":{
      "propertyMaps":[
         {
            "source":{
               "path":"!new('Object')",
               "type":"Object"
            },
            "target":{
               "path":"customer",
               "type":"Object"
            },
            "mappingSpecification":{
               "propertyMaps":[
                  {
                     "source":{
                        "path":"!concat($.firstName, ' ', $.lastName)",
                        "type":"String"
                     },
                     "target":{
                        "path":"name",
                        "type":"String"
                     }
                  }
               ]
            }
         }
      ]
   }
}
```

which will produce:

```json
"customers": [
   {
      "customer": {
         "name": "John Smith"
      }
   },
   {
      "customer": {
         "name": "Simon Pieman"
      }
   }
]
```

note that mapEachMerge won't produce named objects, its merging after all.

## Functions that create arrays

Some functions create an array (or 'collection') as an output, which can then be processed by a mapEach function.
One example is `!unique`. This function will take an array as an input and create a unique array as output. A property is given on which uniqueness will be tested; `!unique( <jsonPathExpr> ,  "uniqueness Property")`.

A typical use of this would be to combine it with a mapEach; `!mapEachMerge( !unique( $.people[*], 'lastName' ))`

Any valid array can be passed to a mapEach from a json path query or a function.

## Absolute and relative queries

As you query the input document, for example selecting an array or object from within the input queries inside the mapping specification for the query are always relative to the object selected.

For the most part this is very convenient but occasionally you'll need to step back outside of the query to pull some data from elsewhere in the input. When you need to do that use the function `!useAbsoluteSource( <jsonPath expr> )` in place of the plain json path. There is also the sister function `!useRelativeSource()`.

### Conditionals

The `!if( expr , then expr, else expr )` function.

`!if( !eq( $.height, $.weight), 'call the doctor', $weight / Pow($.height,2) )  ))`

So if height & weight are equal then call the doctor else calculate BMI.

`!if` is fully nestable so you can evaluate `if then else if...`, etc.

The `!if` function is the only function you can apply to targets, which are normally just dumb recipients of data.

By writing a mapping like:

```json
   "source" : {
         "path" : "!new('Object')",
         "type" : "Object"
   }, 
   "target" : {
         "path" : "!if( @stockSelected , @stock_movement_line_1, !continue() )",
         "type" : "Object"
   }
```

where @stockSelected is a boolean mapping variable. So if stock is selected the we will create the object (it is given a nominal name in this example) else we will continue (and this object is skipped).

So using this we can optionally create sections of json output.

## Type conversion

Type conversion in a mapping will happen automatically so long, of course, as the source data is a valid target type. Inputs must be valid in either the given culture or default culture (en-001). You can set your culture in using the `culture` property.

```json
{
   "input" : {},
   "transformer" : {},
   "culture" : "nn-NO"
}
```

## Temporary Objects

Sometimes it is useful to create temporary data especially when multiple passes of data are required. There is no temporary object or working/scratch data per se, just create an object in the target and use it for transforms that follow.

You will want to use the function !useAbsoluteTarget() to access the data because the default will evaluate your jsonpaths against the relative source data. You can use absolute or relative paths to create your temporary data, it is after all the same as any other target data you are creating. Its probably easier to manage creating absolute in the root of the target.

Finally you can tidy up and remove your temporary objects by using `!delete( $.mypath-to-temp-object )`
`!delete( <string> target path)` only works on target data and it can placed in the target or source map. You cannot delete anything from the source. Source data is immutable.

```json
   "target": {
         "path": "!delete( $.someTempObject )",
         "type": "Object"
   }
```

You can then process this temporary collection from anywhere. It will be at the root of the target, so use `!useAbsoluteTarget( $.#tempData )` to select data from it.

All temporary objects & collections will be automatically removed on completion.

## Built in functions

* !asFunc
* !flatten
* !let
* !set
* !mapEachAdd
* !mapEachMerge
* !groupByAdd
* !groupByMerge
* !copyObject
* !copyObjectWithMapping
* !copyPropsToEach
* !if
* !contact
* !stringContains
* !split
* !splitLeft
* !trim
* !unique
* !new
* !parentName
* !sum
* !null
* !now
* !thisMonth
* !thisYear
* !thisDay
* !exists
* !const
* !not
* !eq
* !ne
* !gt
* !lt
* !or
* !and
* !useAbsoluteSource
* !useRelativeSource
* !useAbsoluteTarget
* !convertToString
* !dateFormat
* !toDate
* !dateAddDays
* !dateAddHours
* !dateAddMonths
* !continue
* !delete

### !asFunc

Inline functions are most useful when data needs to be selected before another operation such as a groupBy. Typically you would expect something like:

!asFunc( !let expr, !let expr, !let expr, expr)

### !flatten

Recursively flattens json objects, preserving arrays but flattening objects in the array. Uses dot notation to name the flattened properties.

### !sum

### !copyObject

Copies (deep clones) either the current object or an object referenced by a jsonpath expression. Optionally can have a list of properties to copy.

`!copyObject()` to make a copy of current object in source.

`!copyObject( 'prop1, prop2, prop3, prop4'  )` to copy named properties only.

`copyObject( $.data.some.json.object )` copies the referenced object.

`copyObject( $.data.some.json.object,  'prop1, prop2, prop3, prop4' )`

### !copyObjectWithMapping

Not quite the same as `!copyObject()`, this function can be used to create a custom object merge with a mapping. It will likely be renamed !mergeObject() in a future release. It takes a source object to copy, which can be provided or defaulted to the current source object, copies the object or only the named properties of the object (referred to as 'copyObject'), and applies a mapping. Optionally, a second object can also be supplied to the mapping (referred to as the 'mergeObject'). The mapping can be used to merge the two objects, returning a new object.

In the mapping refer to the 'copyObject' as `$.copyObject.prop1` and `$.mergeObject.otherProp`.

It has five over loads.

1. `!copyObjectWithMapping( $.object.to.copy , 'prop1, prop2, prop3, prop4', $.data.used.in.mapping)`

2. `!copyObjectWithMapping( $.object.to.copy , $.data.used.in.mapping)`. Same as 1 except copy all properties.

3. `!copyObjectWithMapping( $.object.to.copy , 'prop1, prop2, prop3, prop4')`. Same without any extra data object.

4. `!copyObjectWithMapping( $.object.to.copy)`. Copy the whole object and apply the mapping.

5. `!copyObjectWithMapping( 'prop1, prop2, prop3, prop4' , $.data.used.in.mapping)`. Same but uses the current object instead of a named object.

The function makes the copied object (complete or named properties only) available to the mapping as a local object `$.copy` and the and 


#### Example - using over load 1.

Copy the named properties from `$.object.to.copy` and place in a temporary target object 'copy'. Create a second temporary target object 'data' that references the `$.data.used.in.mapping` object.

In the mapping `copy` & `data` can be accessed to merge the two in any way you see fit.

```json
{
   "source": {
         "path": "!copyObjectWithMapping($.object.to.copy , 'theDate, prop2, prop3, prop4', $.data.used.in.mapping)",
         "type": "Collection"
   },
   "target": {
         "path": "myTarget",
         "type": "Collection"
   },
   "mappingSpecification" : {
      "propertyMaps" : [
         {
            "mapId" : "10 - use copied object to transform the date property name and date format",
            "source" : {
                  "path" : "!dateFormat($.copyObject.theDate, 'yyyy-MM-dd', 'dd.MM.yyyy')",
                  "type" : "String"
            },
            "target" : {
                  "path" : "myDate",
                  "type" : "String"
            }
         },
         {
            "mapId" : "20 - use merge object to get a cost transforming the property name and converting the type",
            "source" : {
                  "path" : "$.mergeOject.amount",
                  "type" : "String"
            },
            "target" : {
                  "path" : "total",
                  "type" : "Decimal"
            }
         }
      ]
   }
}
```

While this kinds of copy can be achieved with mappings alone it saves lots of mapping code where property names don't need to change. It is especially useful when creating temporary working sets of data for processing into the final targets. Whenever two objects need merging this function is potentially useful.

### !copyPropsToEachAdd/Merge

This function does a few things, but one area where it is useful is where access to the parent json object properties is required, something that jsonpath has no built in functionality to handle.

`!copyPropsToEachAdd( 'prop1, prop2, prop3, prop4', $.child.collection[*] )` will take the current object and copy the named properties from it into each element of the given collection, creating a new collection for the target result.

If a mapping specification is provided it will be applied to the new target collection before finally writing to the target. Null/empty objects in the given collection can be optionally skipped `!copyPropsToEachAdd( 'prop1, prop2, prop3, prop4', $.child.collection[*], true/false )` so we only merge and never create new objects where they did not exist in the given collection, the default is `false` (do not skip).

Remember that paths are relative to our position in the source json so with `$.child.collection[*]`, the '$' root is refering to the current object.

The function can be useful in other scenarios where we need to merge objects in two related but separated collections. In this case, use the `!useAbsoluteSource` function like this `!copyPropsToEachAdd( 'prop1, prop2, prop3, prop4', !useAbsoluteSource( $.top.next.relatedCollection[*] ) )`


#### Example : Transform book list to author list

Using the function to get around the parent jsonPath shortcoming. Unlike XPath for XML there is not a mechanism to select properties of a parent object using jsonpath.

In this example we will transform a (very small) book shop database with only 3 books, into an author database. We will change the grammar of the data (creating new objects and new relationships) as well as changing property names and the data itself. We will do this in two steps which in my opinion results in easier to understand and maintain JTL code.

1. Create a temporary working set of data
2. Transform the temporary data into the final database.

##### Book shop Database

```json
{
 "store": {
    "book": [
      {
        "isbn" : "8730-2-44-762112-0",
        "category": "reference",
        "authors": [
          {
            "firstName": "Nigel",
            "lastName": "Rees"
          },
          {
            "firstName": "Evelyn",
            "lastName": "Waugh"
          }
        ],
        "title": "Sayings of the Century",
        "price": 8.95
      },
      {
        "isbn" : "0304368032",
        "category": "reference",
        "authors": [
          {
            "firstName": "Nigel",
            "lastName": "Rees"
          }
        ],
        "title": "I Told You I Was Sick ",
        "price": 10.95
      }
    ]
}
```

##### Target author database

```json
{
   "writers" : [
      {
         "name" : "Rees, Nigel",
         "books" : [
            {
               "isbn" : "8730-2-44-762112-0",
               "category": "reference",
               "title": "Sayings of the Century",
               "price": 8.95
            },
            {
               "isbn" : "0304368032",
               "category": "reference",
               "title": "I Told You I Was Sick ",
               "price": 10.95
            }
         ]
      },
      {
         "name" : "Waugh, Evelyn",
         "books" : [
            {
               "isbn" : "8730-2-44-762112-0",
               "title": "Sayings of the Century",
               "price": 8.95
            }
         ]
      }
   ]
}
```

Same data but a different relationship. This kind of relationship re-arranging transformation occurs remarkably often where we see two related systems that offer different views of the world. e-shops and accounting for example; both require tracking purchases but for different reasons, resulting in different data grammars and syntax.

The `!copyPropsToEach` comes in very useful in these scenarios by helping to create a list of objects we can use as a temporary working data set, which can then be processed into the relationships we want.

`!copyPropsToEach( 'isbn, category, title, price', $.authors[*] )` will neatly copy each of the named properties of the current book object into each of the author objects and spit out the authors into a collection. The result has a collection of single objects each with all the data we need from both parent and child - `!copyPropsToEach` has removed the need for a parent reference in jsonpath.

```json
[
   {
      "firstName" : "Nigel",
      "lastName": "Rees",
      "isbn" : "8730-2-44-762112-0",
      "category": "reference",
      "title": "Sayings of the Century",
      "price": 8.95
   },
   {  
      "firstName" : "Nigel",
      "lastName": "Rees",
      "isbn" : "0304368032",
      "category": "reference",
      "title": "I Told You I Was Sick ",
      "price": 10.95
   }
]

```

We do that for each book in the source data which gets us pretty close to the final target. By applying `!unique()` to the resulting array we select only the 1st author record and use that to create the top level author object. Then we can select into our temporary author collection to select all the records for that author and use that to create the book entries for the author. The final JTL will look something like this:

```json
{
    "mappingSpecification": {
        "propertyMaps": [
            {
                "mapId": "100 - create flat author list",
                "source": {
                    "path": "!mapEachMerge( $.store.books[*] )",
                    "type": "Collection"
                },
                "target": {
                    "path": "authorBooks",
                    "type": "Collection"
                },
                "mappingSpecification": {
                    "propertyMaps": [
                        {
                            "mapId": "200-create working set",
                            "source": {
                                "path": "!copyPropsToEachAdd( 'isbn, category, title, price', $.authors[*] )",
                                "type": "Collection"
                            },
                            "target": {
                                "path": "_",
                                "type": "Collection"
                            }
                        }
                    ]
                }
            },
            {
               "mapId": "400-process each author to final object",
               "source": {
                  "path": "!mapEachMerge( !unique( !useAbsoluteTarget( $.authorBooks[*] ), \"$.lastName\" ) )",
                  "type": "Collection"
               },
               "target": {
                  "path": "writers",
                  "type": "Collection"
               },
               "mappingSpecification": {
                  "propertyMaps": [
                     {
                           "mapId": "600",
                           "source": {
                              "path": "!new('Object')",
                              "type": "Object"
                           },
                           "target": {
                              "path": "author",
                              "type": "Object"
                           },
                           "mappingSpecification": {
                              "propertyMaps": [
                                 {
                                       "mapId": "800",
                                       "source": {
                                          "path": "!concat( $.lastName, ', ', $.firstName)",
                                          "type": "String"
                                       },
                                       "target": {
                                          "path": "name",
                                          "type": "String"
                                       }
                                 }
                              ]
                           }
                     }  NEEDS FINISHING FOR AUTHORS BOOKS
                  ]
               }
            }
        ]
    }
}
```

"map 100 - create flat author list" selects each of the books from the store.books collection and for each of them copies the named book properties into each object in the collection `$.authors[*]`. The `!copyPropsToEachAdd` has no mapping specification to apply so the merged array is passed back up to "100 - create flat author list" to be added into the 'authorbooks' target. At this point we have our temporary working collection called 'authorbooks' which we can process to the final target.

"400-process each author to final object" takes care of this. Notice how the `!mapEachMerge` makes use of `!useAbsoluteTarget` function to switch the data we are selecting to use targets we have already written, instead of the default relative source data. We add a `!unique` to select only the first 'authorbook' found for each author last name. In other words we select each author once only, and we will pass that down to the mapping specification to build out the finished author object, with a list of books they wrote.

NEEDS FINISHING FOR AUTHORS BOOKS



## Reference

### unique

`!unique( <collection> to search, <string> json path of property to use for uniqueness)`

## Advanced Example - multiple passes, uniqueness, grouping and summing

### The problem:

Source data that looks like:

```json
   {
      "timestamp": "2020-01-30T19:01:39.281+0000",
      "amount": -70,
      "originatorTransactionType": "CARD_PAYMENT_FEE",
      "originatingTransactionUuid": "f1432cfa-4392-11ea-86f2-e092f017ced5"
   },
   {
      "timestamp": "2020-01-30T19:01:39.281+0000",
      "amount": 4000,
      "originatorTransactionType": "CARD_PAYMENT",
      "originatingTransactionUuid": "f1432cfa-4392-11ea-86f2-e092f017ced5"
   },
   {
      "timestamp": "2020-01-28T11:51:11.406+0000",
      "amount": -73687,
      "originatorTransactionType": "PAYOUT",
      "originatingTransactionUuid": "85608f06-41c0-11ea-afa0-6118fb53485f"
   },
   {}
   ...
```

Transactions are time stamped and we need to group and total transactions by date and type `originatorTransactionType` . We need to be able to easily add new types and create extra properties.

The transactions for a day will be transformed into one daily accounting 'voucher' that will have a header and a collection of voucher lines, one for each transaction type, plus an extra balancing or summary line.

The first problem we can see straight away is how to select transactions for one day where the timestamps are not by day but by millisecond. There are a couple of options here. But remembering that we are doing this declaratively (like excel if you're not sure what means) will help us. Programming declaratively is more akin to stating what you want rather than how you want to do it.

So, I think it would be easier to deal with this if the source data looked a little different. So I want to do a 2-pass transformation. The 1st pass will be to remove data I'm not interested in and change the date format to something a little simpler to deal with. Make transactions that look like this:

```json
   {
      "transactionDate": "2020.01.31",
      "amount": -2625.0,
      "originatorTransactionType": "CARD_PAYMENT_FEE"
   }
   ```
That's pretty easy to do with a !mapEachAdd function that creates a new collection from all the transactions in the source.

```json
   {
      "mapId": "100 - 1st pass",
      "source": {
            "path": "!mapEachAdd( $.data.transactions[*])",
            "type": "Collection"
      },
      "target": {
            "path": "transactionsDateformat",
            "type": "Collection"
      }
   }
```

I'll give the !mapEachAdd function a `mappingSpecification` that creates the properties we want to see in each object of the collection.

```json
   "mappingSpecification": {
         "mappingVariables": null,
         "propertyMaps": [
            {
               "mapId": "200 - take the timestamp and reformat it.",
               "source": {
                     "path": "!dateFormat( $.timestamp, 'yyyy-MM-dd'T'HH:mm:ss.FFFK', 'yyyy.MM.dd' )",
                     "type": "String"
               },
               "target": {
                     "path": "transactionDate",
                     "type": "String"
               }
            },
            {
               "mapId": "300",
               "source": {
                     "path": "$.amount",
                     "type": "Decimal"
               },
               "target": {
                     "path": "amount",
                     "type": "Decimal"
               }
            },
            {
               "mapId": "400",
               "source": {
                     "path": "$.originatorTransactionType",
                     "type": "String"
               },
               "target": {
                     "path": "originatorTransactionType",
                     "type": "String"
               }
            }
         ]
   }
```

That will create for us a new array in the target document called `transactionsDateformat` where each object in the array has the 3 properties defined here.

Now we can use that array to easily select out transactions for only one day. Whats more I'm going to do that and create a temporary array of unique days that I can iterate over to create the daily vouchers. Like this:

```json
{
   "mapId": "1000 - process 2nd pass, create vouchers",
   "source": {
         "path": "!mapEachAdd( !useAbsoluteTarget( !unique( $.transactionsDateformat[*],  \"$.transactionDate\" )) )",
         "type": "Collection"
   },
   "target": {
         "path": "manualJournalVouchers",
         "type": "Collection"
   }
}
```

With some nested functions to achieve this; So I want to mapEach unique date - i.e. create a voucher for each day and only one voucher for a day. The `!unique` function handles this well. But normally we would select data from the source document but in this case we have created a working collection in our target so the `!unique` is combined with `!useAbsoluteTarget` to run the query against our target document. `!unique` will return the first object it finds when there are more than one sharing the same `transactionDate`, creating a unique array. We are going to map that array to the final vouchers array.

So for each day in the new unique array we will select all the transactions from the working array we already created called `transactionDateFormat`. We do that by using the json path select and another !mapEach.

```json
{
   "mapId": "1400",
   "source": {
         "path": "!mapEachMerge( !useAbsoluteTarget( $.transactionsDateformat[?(@.transactionDate == $voucherDate)]))",
         "type": "Collection"
   },
   "target": {
         "path": "lines",
         "type": "Collection"
   }
}
```

put in mapping specification for this map each.
explain temp variable of !let
explain the !asFunc
finally add in the groupBy


$voucherDate is just a variable set to the current unique date. That will select all the transaction lines from the working table for a particular day. What we need to do now is group those by transaction type. We can use the `!groupByAdd` function like this : 

```jtl
!groupByAdd( $dayTransactions,
            [\"date\", \"currencyCode\", \"accountCode\", \"description\"],
            \"new( Key.date, Key.currencyCode, Key.accountCode, Key.description, Sum(Convert.ToDecimal(it[\"amount\"])) as amount)\"))
```

of course normally we cannot split the line in json but I do so here for the sake of clarity. We are saying to group the.....
