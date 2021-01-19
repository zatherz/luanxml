LuaNXML
===

LuaNXML is a Lua port of [NXML](https://github.com/XWitchProject/NXML).

NXML is a pseudo-XML parser that can read and write the oddly formatted and often invalid XML files from [Noita](https://noitagame.com). NXML (the C#/.NET version linked before) is itself a port of the XML parser from [Poro](https://github.com/gummikana/poro), which is used by the custom Falling Everything engine on which Noita runs.

In short, LuaNXML is an XML parser that is 100% equivalent to Noita's parser. As opposed to something like [xml2lua](https://github.com/manoelcampos/xml2lua), LuaNXML is just as non-conformant to the XML specification as Noita itself. It can also produce semantically equivalent output in the form of a string.

# Example

The code below enables all mods in the `mod_config.xml` file by setting the `enabled` attribute on all children to `true`. Boolean values are automatically converted to Noita XML style `"1"` or `"0"` values.

```lua
local nxml = require("nxml")

local f = io.open("save00/mod_config.xml", "r")
local xml = nxml.parse(f:read("*a"))
f:close()

for elem in xml:each_child() do
    elem.attr.enabled = true
end

f = io.open("save00/mod_config.xml", "w")
f:write(tostring(xml))
f:close()
```

# API

LuaNXML's API is a bit more object oriented and abstract than something like xml2lua. This is to avoid any unexpected surprises like child elements suddenly becoming tables if there's more than one of them or being unable to have attributes with certain names.

* **table** `nxml`
    * **function** `.parse(data : string)` **returns** `nxml.element`

      parses `data` into an `nxml.element` type table  
      `data` must be a single element (which may have children), like:  
      ```xml
      <foo />
      ```
      or
      ```xml
      <foo>
        <bar>
          <baz/>
        </bar>
      </foo>
      ```

    * **function** `.parse_many(data : string)` **returns** `table[nxml.element]`

      parses `data` into a a table of `nxml.element`s  
      the difference between `.parse_many` and `.parse` is that `.parse_many` can read multiple elements written one after another, like this:  
      ```xml
      <foo />
      <bar />
      <baz />
      ```  
      and it will spit out a table of all root elements

    * **function** `.tostring(elem : nxml.element, [packed : boolean, [indent_char : string, [cur_indent : string]]])` **returns** `string`

      serializes the `nxml.element` object into a valid Noita XML string that can be read by the game  
      if `packed` is set to `true`, the string is compressed into a single line - otherwise, line breaks and indents are used  
      `indent_char` determines what character to use for indentation - it is equal to `"\t"` (a single tab) if not passed  
      `cur_indent` determines the string to prefix any child element indentation - it is equal to `""` (an empty string) if not passed  
      this function is used by the `nxml.element` `__tostring` metamethod for convenience, but the full form allows more fine tuning of the output

    * **function** `.new_element(name : string[, attr : table{string => any}])` **returns** `nxml.element`

      creates a new element with the name passed in `name`  
      if `attr` is not nil, the `.attr` field of the new `nxml.element` is initialized with the passed value

* **table type** `nxml.element`
    * **field** `.name` **of type** `string`

      contains the name of the element (e.g. `<Foo a="1" />` has the name `Foo`)

    * **field** `.children` **of type** `table[nxml.element]`

      contains a table of all children (see: `:each_child()`)

    * **field** `.attr` **of type** `table{string => any}`

      contains a table of the element's attributes  
      `nxml.parse` will always return elements where all the values inside `.attr` are of type `string`, but it is perfectly fine to assign values of different types like numbers or booleans  
      booleans will be converted to `"1"` or `"0"` when generating the XML string with `nxml.string`/`tostring` - all other types are simply ran through `tostring` to produce the serialized value

    * **field** `.content` **of type** `table{string}`

      contains a list of content tokens within the element  
      as is the case in Noita, text content inside elements is parsed on a token-by-token basis, and this table contains the list of those tokens  
      may be `nil` or contain 0 elements, in both cases that means the text element has no content  
      example: `<foo>foo/bar</foo>` and `<foo> foo / bar </foo>` will both result in this field containing the strings `foo`, `/` and `bar`  
      see: `:text()`

    * **function** `:text()` **returns** `string`

      returns the text content of this element as a single string  
      text that is not punctuation (i.e. is not `<`, `>`, `/` and `=`) is joined with a single space, punctuation is joined without any separators  
      example: `<foo> foo   bar </foo>` will result in `foo bar`, while `<foo>foo/bar/baz</foo>` will result in `foo/bar/baz`  
      see: `.content`

    * **iterator** `:each_child()` **yields** `nxml.element`

      iterates every child element  
      used like this: `for child in elem:each_child() ...`

    * **iterator** `:each_of(element_name : string)` **yields** `nxml.element`

      iterates every child element with the same name as `element_name`  
      used like this: `for child in elem:each_of("Foo") ...`

    * **function** `:first_of(element_name : string)` **returns** `nxml.element`

      returns the first child element with the name of `element_name`, or `nil` if no such element exists

    * **function** `:all_of(element_name : string)` **returns** `table[nxml.element]`

      returns a table with every child element with the same name as `element_name`  

    * **function** `:add_child(element : nxml.element)`

      adds a child element

    * **function** `:add_children(elements : table[nxml.element])`

      adds all elements from the table as children

    * **function** `:remove_child(element : nxml.element)`

      removes the passed element from this element's list of children

    * **function** `:remove_child_at(index : number)`

      removes the element at index `index` from this element's list of children

    * **function** `:clear_children()`

      removes all children of this element

    * **function** `:clear_attrs()`

      removes all attributes of this element

    * **metamethod** `__tostring`  

      using the Lua `tostring` function on an `nxml.element` will produce a valid Noita XML string  
      see: `nxml.tostring`

      

## LuaJIT FFI

If the `require` function is available and the `ffi` module can be loaded (which is the case in a Noita mod if your `mod.xml` has `request_no_api_restrictions="1"` in it), minimal optimizations in string handling are used (which should result in less intermediate strings). The output and behavior is equivalent to when `ffi` is not available.
