![abaplint](https://github.com/sbcgua/abap-string-map/workflows/abaplint/badge.svg)

# abap json (ajson)

Yet another json parser/serializer for ABAP. Should work on 7.31, maybe on 7.02 but with json sxml parser installed.

Features:
- parse into a flexible form, not fixed to any predefined data structure, allowing to change the parsed data and slice subsections of it
  - slicing can be particulary useful for REST header separation e.g. `{ success: 1, error: "", payload: {...} }` where `success` and Ko is processed in one layer of your application and payload in another (and can differ)
- allows conversion to fixed abap structures/tables (`to_abap`)
- convenient interface to manipulate the data (`set( value )`, `set( structure )`, `set( table )`, `set( another_instance_of_ajson )`)
- seralization to string (TODO)

Installed using [abapGit](https://github.com/larshp/abapGit)

## Examples and documentation

The class `zcl_ajson` implements 2 interfaces:
- `zif_ajson_reader` - used to access items of the json data
- `zif_ajson_writer` - used to set items of the json data

### Instantiating and basics

- To parse existing json data - call `zcl_ajson=>parse( lv_json_string )`
- To create a new empty json instance (to set values and serialize) - call `zcl_ajson=>create_empty( lv_json_string )`
- Json attributes are addressed by path in form of e.g. `/a/b/c` addresses `{ a: { b: { c: "this value !" } } }`
- Array items addressed with index starting from 1: `/tab/2/val` -> `{ tab: [ {...}, { val: "this value !" } ] }`

### zif_ajson_reader

Assuming original json is

```json
{
  "success": 1,
  "error": "",
  "payload": {
    "text": "hello",
    "num": 123,
    "bool": true,
    "false": false,
    "null": null,
    "table": [
      "abc",
      "def"
    ]
  }
}
```

```abap
data r type ref to zif_ajson_reader.
r = zcl_ajson=>parse( lv_json_string_from_above ).

r->exists( '/success' ).              " returns abap_true

r->value( '/success' ).               " returns "1"
r->value_integer( '/success' ).       " returns 1 (number)
r->value_boolean( '/success' ).       " returns "X" (abap_true - because not empty)

r->value( '/payload/bool' ).          " returns "true"
r->value_boolean( '/payload/bool' ).  " returns "X" (abap_true)

r->value( '/payload/false' ).         " returns "false"
r->value_boolean( '/payload/false' ). " returns "" (abap_false)

r->value( '/payload/null' ).          " returns "null"
r->value_string( '/payload/null' ).   " returns "" (empty string)

r->members( '/' ).                    " returns table of "success", "error", "payload"

" Slice returns zif_ajson_reader instance but "payload" becomes root
" Useful to process API responses with unified wrappers
data payload type ref to zif_ajson_reader.
payload = r->slice( '/payload' ). 

" converting to abap structure
data:
  begin of ls_payload,
    text type string,
    num type i,
    bool type abap_bool,
    false type abap_bool,
    null type string,
    table type string_table, " Array !
  end of ls_payload.

payload->to_abap( importing ev_container = ls_payload ).
```
### zif_ajson_writer

```abap
data w type ref to zif_ajson_writer.
w = zcl_ajson=>create_empty( ).

" Set value
" Results in { a: { b: { num: 123, str: "hello", bool: true } } }
" The intermediary path is auto created, value type auto detected
w->set(
  iv_path = '/a/b/num'
  iv_val  = 123 ).
w->set(
  iv_path = '/a/b/str'
  iv_val  = 'hello' ).
w->set(
  iv_path = '/a/b/bool'
  iv_val  = abap_true ).

" Important values and whole branches are rewritten
" { a: { b: 0 } } - the old "b" completely deleted
w->set(
  iv_path = '/a/b'
  iv_val  = 0 ).

" Items can be deleted explicitly
w->delete( '/a/b' ). " => { a: { } }

" Or completely cleared
w->clear( ).

" Set object
" Results in { a: { b: { payload: { text: ..., num: ... } } } }
data:
  begin of ls_payload,
    text type string,
    num type i,
  end of ls_payload.
w->set(
  iv_path = '/a/b/payload'
  iv_val  = ls_payload ).

" Set arrays
" Results in: { array: [ "abc", "efg" ] }
" Tables of structures, of tables, and other deep objects are supported as well
data tab type string_table.
append 'abc' to tab.
append 'efg' to tab.
w->set(
  iv_path = '/array'
  iv_val  = tab ).

" Fill arrays item by item
" Different types ? no problem
w->push(
  iv_path = '/array'
  iv_val  = 1 ).
" => { array: [ "abc", "efg", 1 ] }

w->push(
  iv_path = '/array'
  iv_val  = ls_payload ).
" => { array: [ "abc", "efg", 1, { text: ..., num: ... } ] }

" Push verifies that the path item exists and is array
" it does NOT auto create path like "set"
" to explicitly create an empty array use "touch_array"
w->touch_array( '/array2' ).
" => { array: ..., array2: [] }
```

## Known issues

- removing an array item in the middle of array will not renumber the items

## References

- Forked from [here](https://github.com/abaplint/abaplint-abap-backend) originally
