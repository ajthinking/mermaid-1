# Adding a New Diagram/Chart 📊

### Step 1: Grammar & Parsing


#### Grammar

This would be to define a jison grammar for the new diagram type. That should start with a way to identify that the text in the mermaid tag is a diagram of that type. Create a new folder under diagrams for your new diagram type and a parser folder in it. This leads us to step 2.

For instance:

* the flowchart starts with the keyword graph.
* the sequence diagram starts with the keyword sequenceDiagram

#### Store data found during parsing

There are some jison specific sub steps here where the parser stores the data encountered when parsing the diagram, this data is later used by the renderer. You can during the parsing call a object provided to the parser by the user of the parser. This object can be called during parsing for storing data.

```jison
statement
	: 'participant' actor  { $$='actor'; }
	| signal               { $$='signal'; }
	| note_statement       { $$='note';  }
	| 'title' message      { yy.setTitle($2);  }
	;
```

In the extract of the grammar above, it is defined that a call to the setTitle method in the data object will be done when parsing and the title keyword is encountered.

```tip
Make sure that the `parseError` function for the parser is defined and calling `mermaid.parseError`. This way a common way of detecting parse errors is provided for the end-user.
```

For more info look in the example diagram type:

The `yy` object has the following function:

```javascript
exports.parseError = function(err, hash){
   mermaid.parseError(err, hash)
};
```

when parsing the `yy` object is initialized as per below:

```javascript
var parser
parser = exampleParser.parser
parser.yy = db
```


### Step 2: Rendering

Write a renderer that given the data found during parsing renders the diagram. To look at an example look at sequenceRenderer.js rather then the flowchart renderer as this is a more generic example.

Place the renderer in the diagram folder.


### Step 3: Detection of the new diagram type

The second thing to do is to add the capability to detect the new new diagram to type to the detectType in utils.js. The detection should return a key for the new diagram type.


### Step 4: The final piece - triggering the rendering

At this point when mermaid is trying to render the diagram, it will detect it as being of the new type but there will be no match when trying to render the diagram. To fix this add a new case in the switch statement in main.js:init this should match the diagram type returned from step #2. The code in this new case statement should call the renderer for the diagram type with the data found by the parser as an argument.


## Usage of the parser as a separate module


### Setup

```javascript
var graph = require('./graphDb')
var flow = require('./parser/flow')
flow.parser.yy = graph
```


### Parsing

```javascript
flow.parser.parse(text)
```


### Data extraction

```javascript
graph.getDirection()
graph.getVertices()
graph.getEdges()
```

The parser is also exposed in the mermaid api by calling:

```javascript
var parser = mermaid.getParser()
```

Note that the parse needs a graph object to store the data as per:

```javascript
flow.parser.yy = graph
```

Look at `graphDb.js` for more details on that object.

## Layout

If you are using a dagre based layout, please use flowchart-v2 as a template and by doing that you will be using dagre-wrapper instead of dagreD3 which we are m igrating away from.

### Common parts of a diagram

There are a few features that are common between the different types of diagrams. We try to standardize the diagrams that work as similar as possible for the end user. The commonalities are:

* Directives, a way of modifying the diagram configuration from within the diagram code.
* Accessibility, a way for an author to provide additional information like titles and descriptions to people accessing a text with diagrams using a screen reader.
* Themes, there is a common way to modify the styling of diagrams in Mermaid.
* Comments should follow mermaid standards

Here some pointers on how to handle these different areas.

#### [Directives](./directives.md)

Here is example handling from flowcharts:
Jison:
```jison

/* lexial grammar */
%lex
%x open_directive
%x type_directive
%x arg_directive
%x close_directive

\%\%\{                                                          { this.begin('open_directive'); return 'open_directive'; }
<open_directive>((?:(?!\}\%\%)[^:.])*)                          { this.begin('type_directive'); return 'type_directive'; }
<type_directive>":"                                             { this.popState(); this.begin('arg_directive'); return ':'; }
<type_directive,arg_directive>\}\%\%                            { this.popState(); this.popState(); return 'close_directive'; }
<arg_directive>((?:(?!\}\%\%).|\n)*)                            return 'arg_directive';

/* language grammar */

/* ... */

directive
  : openDirective typeDirective closeDirective separator
  | openDirective typeDirective ':' argDirective closeDirective separator
  ;

openDirective
  : open_directive { yy.parseDirective('%%{', 'open_directive'); }
  ;

typeDirective
  : type_directive { yy.parseDirective($1, 'type_directive'); }
  ;

argDirective
  : arg_directive { $1 = $1.trim().replace(/'/g, '"'); yy.parseDirective($1, 'arg_directive'); }
  ;

closeDirective
  : close_directive { yy.parseDirective('}%%', 'close_directive', 'flowchart'); }
  ;
```

It is probably a good idea to keep the handling similar to this in your new diagram. The parseDirective function is provided by the mermaidAPI.

## Accessibility

The syntax for adding title and description looks like this:
```
accTitle: The title
accDescr: The dsscription

accDescr {
	Syntax for a description text
	written on multiple lines.
}
```

In a similar way to the directives the jison syntax are quite similar between the diagrams.

```jison
* lexical grammar */
%lex
%x acc_title
%x acc_descr
%x acc_descr_multiline

%%
accTitle\s*":"\s*                                { this.begin("acc_title");return 'acc_title'; }
<acc_title>(?!\n|;|#)*[^\n]*                     { this.popState(); return "acc_title_value"; }
accDescr\s*":"\s*                                { this.begin("acc_descr");return 'acc_descr'; }
<acc_descr>(?!\n|;|#)*[^\n]*                     { this.popState(); return "acc_descr_value"; }
accDescr\s*"{"\s*                                { this.begin("acc_descr_multiline");}
<acc_descr_multiline>[\}]                        { this.popState(); }
<acc_descr_multiline>[^\}]*                      return "acc_descr_multiline_value";

statement
    : acc_title acc_title_value  { $$=$2.trim();yy.setTitle($$); }
    | acc_descr acc_descr_value  { $$=$2.trim();yy.setAccDescription($$); }
    | acc_descr_multiline_value { $$=$1.trim();yy.setAccDescription($$); }
```

The functions for setting title and description are provided by a common module. This is the import from flowDb.js:

```
import {
  setAccTitle,
  getAccTitle,
  getAccDescription,
  setAccDescription,
  clear as commonClear,
} from '../../commonDb';
```

For rendering the accessibility tags you have again an existing function you can use.

**In the renderer:**
```js
import addSVGAccessibilityFields from '../../accessibility';

/* ... */

// Adds title and description to the flow chart
addSVGAccessibilityFields(parser.yy, svg, id);
```

## Theming

Mermaid supports themes and has an integrated theming engine. You can read more about how the themes can be used [in the docs](./theming.md).

When adding themes to a diagram it comes down to a few important locations in the code.

The entry point for the styling engine is in **src/styles.js**. The getStyles function will be called by Mermaid when the styles are being applied to the diagram.

This function will in turn call a function *your diagram should provide* returning the css for the new diagram. The diagram specific, also which is commonly also called getStyles and located in the folder for your diagram under src/diagrams and should be named styles.js. The getStyles function will be called with the theme options as an argument like in the following example:

```js
const getStyles = (options) =>
    `
    .line {
      stroke-width: 1;
      stroke: ${options.lineColor};
      stroke-dasharray: 2;
    }
    // ...
    `;
```

Note that you need to provide your function to the main getStyles by adding it into the themes object in  **src/styles.js** like in the xyzDiagram in the provided example:

```js
const themes = {
    flowchart,
    'flowchart-v2': flowchart,
    sequence,
    xyzDiagram,
    //...
};
```

The actual options and values for the colors are defined in **src/theme/theme-[xyz].js**. If you provide the options your diagram needs in the existing theme files then the theming will work smoothly without hiccups.

