Building JSON parser using FsLex and FsYacc
===========================================

Content:

1. [Introduction](#introduction)
2. [Syntax tree](#syntaxtree)
3. [Parser](#parser)
4. [Lexer](#lexer)
5. [Program](#program)


1 Introduction
----------------
FsLexYacc is a solution which allows you to define and generate lexer and parser. It's made of two parts FsLex (lexer generator) and FsYacc (parser generator). Parsing is a two phase process. In the first phase lexer is analyzing text and creates a stream of tokens (or token list). In the second phase parser is going through tokens in the stream and generates output (syntax tree).

FsYacc (parser generator) is decoupled from FsLex (lexer generator) and can accept a token stream from any lexer. Generated parser will require a function which can translate tokens generated by lexer into tokens configured in parser. To avoid additional pointless work if you use FsYacc with FsLex define parser first. This way FsYacc will generate an union type defining all required tokens. Then you can use this union type to generate tokens inside lexer. So despite fact that lexing happens before parsing we will define parser first. It means that in your F# project generated parser must be first on the files list before lexer.
2 Syntax tree
-----------------
Create a new F# library project and install FsLexYacc package:
    PM> Install-Package FsLexYacc
We will start by describing the syntax tree. It will be result of parsing a text. Add a new file called ``JsonValue.fs`` and paste below union type definition:

    type JsonValue = 
      | Assoc of (string * JsonValue) list
      | Bool of bool
      | Float of float
      | Int of int
      | List of JsonValue list
      | Null
      | String of string


Nothing fancy here. Assoc is simply an object which contains a list of properties (pair where ``fst`` is a property name and ``snd`` is a value).
``List`` is for JSON arrays. We also have numbers, bool, string and null. It's enough to describe a valid JSON.

3 Parser definition
----------------------

In your project root directory (not solution root) create a file called ``Parser.fsy``. Now edit project file (.fsproj ).  Find  the line where JsonValue.fs is included, than add this element this xml:

    <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
    </FsYacc>

Outside ItemGroup element add this lines:

    <Import Project="$(FSharpTargetsPath)" Condition="Exists('$(FSharpTargetsPath)')" />
    <Import Project="..\..\bin\Debug\FsLexYacc.targets" />

The whole thing should be like this:

    ...
    <Import Project="$(FSharpTargetsPath)" Condition="Exists('$(FSharpTargetsPath)')" />
    <Import Project="..\..\bin\Debug\FsLexYacc.targets" />
    <ItemGroup>
        <Compile Include="JsonValue.fs" />
        <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
        </FsYacc>
        ...
    <ItemGroup>
    ...

Reload/Open the project and add below code to Parser.fsy:

    //This parser has been writen with help of "Real world OCaml" book By Yaron Minsky, Anil Madhavapeddy, Jason Hickey (chapter 16)
    %{
    open JsonParsing
    %}
    
    %start start
    
    %token <int> INT
    %token <float> FLOAT
    %token <string> ID
    %token <string> STRING
    %token TRUE
    %token FALSE
    %token NULL
    %token LEFT_BRACE
    %token RIGHT_BRACE
    %token LEFT_BRACK
    %token RIGHT_BRACK
    %token COLON
    %token COMMA
    %token EOF
    
    %type <JsonParsing.JsonValue option> start
    
    %%
    
    start: prog { $1 }
    
    prog:
      | EOF { None }
      | value { Some $1 }
    
    value:
      | LEFT_BRACE object_fields RIGHT_BRACE { Assoc $2 }
      | LEFT_BRACK array_values RIGHT_BRACK { List $2 }
      | STRING { String $1 }
      | INT { Int $1 }
      | FLOAT { Float $1 }
      | TRUE { Bool true }
      | FALSE { Bool false }
      | NULL { Null }
    
    object_fields: rev_object_fields { List.rev $1 };
    
    rev_object_fields:
      | { [] }
      | STRING COLON value { [($1,$3)] }
      | rev_object_fields COMMA STRING COLON value { ($3, $5) :: $1 }
    
    array_values:
      | { [] }
      | rev_values { List.rev $1 }
    
    rev_values:
      | value { [$1] }
      | rev_values COMMA value { $3 :: $1 }
This file is describing parsing rules and tokens. When you build project, in the project root directory a new file will be created called ``Parser.fs``. Include it in your project.

Lets look closer at parser definition (``Parser.fsy)``. At the very top of the file there is a section for ``open`` statements. You can open any namespace or module you want. We need only ``JsonParsing`` where we defined ``JsonValux`` structure. All open statements should be between`` %{`` and ``%}. ``

Next in line 6 we state what is an entry rule for our parser. ``start ``will be the name of the rule which will be exposed by parsing module. It has to correspond to a name of rule (in our case line 27).

Lines 8-21 define list of tokens. INT, FLOAT, STRING carry a value with them.

Then in line 23 we define what will be the result of parsing. In our case it will be JSON syntax tree defined by means of ``JsonValue``. The type is actually ``JsonValue option``. For empty text/file parser will return ``NONE``.

Everything that follows ``%%`` are parsing rules. Rules are made of list of productions. You can reference one rule from the other which will allow you to define nested parsers (we will use it for objects and lists).

Our entry point (line 27) simply calls ``prog ``rule and returns it's result. Everything between curly braces is an ordinary F# code.

``prog ``(line 22) has two productions. First says that if the first token is ``EOF ``then return ``None. ``The second  rule says execute ``value ``rules and return ``Some ``containing it's result. Productions are processed one after another from top to bottom.

On line 33 we define main rule which will parse JSON values. Each rule has a name and list of productions. Each production starts with tokens pattern  and contains a result block ``{} `` which says what should be returned if this pattern is matched. Result block contains ordinary F# code.

Rules can reference each other. Like at line 34 where the pattern says match a left brace, whatever will be matched by ``object_fields rule`` and a right brace. Now what is this ``Assoc $2. ``Assoc is JsonValue union type that we create and $2 corresponds to value matched at second position in pattern which in this case is a list of properties.

If you look at ``object_fields ``rule, you'll notice it actually calls ``rev_object_fields`` and reverses order of results.  ``rev_object_fields`` is trying to match list of properties but it will collect them in wrong order. First production says that if token stream is empty (there is nothing between brace) then return empty list. We match an empty stream by not providing any pattern. Second rule is more interesting. It says that if we encounter a string then a colon and anything that matches the ``value`` rule we should return one element list containing a pair. First element in pair is matched string and second is matched value. This production will be used for objects that have one element or for the last property on the list (there is no comma at the end). Third rule contains two references to other rules. First we match any list of values then a comma, a string, a colon and ``value``. And the result is a list where head is matched property (tokens on position 3-5) and tail is made of other matched properties. This production is for all properties which are followed by comma. Rules for arrays are very similar (even simpler).

That's it we can now start building lexer. You should be able to compile the project and see generated parser in ``Parser.fs``. It looks ugly but fortunately we will always deal only with grammar description.
4 Lexer
-----------

In your project root directory (not solution root) create a file called ``Lexer.fsl``. Now edit the project file (.fsproj ).  Find the line where ``JsonValue.fs`` is included then add this elment:

    <FsLex Include="Lexer.fsl">
        <OtherFlags>--unicode</OtherFlags>
    </FsLex>

The whole thing should be like this:

    ...
    <Import Project="$(FSharpTargetsPath)" Condition="Exists('$(FSharpTargetsPath)')" />
    <Import Project="..\..\bin\Debug\FsLexYacc.targets" />
    <ItemGroup>
      <Compile Include="JsonValue.fs" />
      <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
      </FsYacc>
      <FsLex Include="Lexer.fsl">
        <OtherFlags>--unicode</OtherFlags>
      </FsLex>
    ...

Reload/Open project and add below code to ``Lexer.fsl:``

    //This lexer has been writen with help of "Real world OCaml" book By Yaron Minsky, Anil Madhavapeddy, Jason Hickey (chapter 16)
    {
    
    module Lexer
    
    open FSharp.Text.Lexing
    open System
    open Parser
    
    exception SyntaxError of string
    
    let lexeme = LexBuffer.LexemeString
    
    let newline (lexbuf: LexBuffer<_>) = 
      lexbuf.StartPos <- lexbuf.StartPos.NextLine
    }
    
    let int = ['-' '+']? ['0'-'9']+
    let digit = ['0'-'9']
    let frac = '.' digit*
    let exp = ['e' 'E'] ['-' '+']? digit+
    let float = '-'? digit* frac? exp?
    
    let white = [' ' '\t']+
    let newline = '\r' | '\n' | "\r\n"
    
    rule read =
      parse
      | white    { read lexbuf }
      | newline  { newline lexbuf; read lexbuf }
      | int      { INT (int (lexeme lexbuf)) }
      | float    { FLOAT (float (lexeme lexbuf)) }
      | "true"   { TRUE }
      | "false"  { FALSE }
      | "null"   { NULL }
      | '"'      { read_string "" false lexbuf } 
      | '{'      { LEFT_BRACE }
      | '}'      { RIGHT_BRACE }
      | '['      { LEFT_BRACK }
      | ']'      { RIGHT_BRACK }
      | ':'      { COLON }
      | ','      { COMMA }
      | eof      { EOF }
      | _ { raise (Exception (sprintf "SyntaxError: Unexpected char: '%s' Line: %d Column: %d" (lexeme lexbuf) (lexbuf.StartPos.Line+1) lexbuf.StartPos.Column)) }
    
    
    and read_string str ignorequote =
      parse
      | '"'           { if ignorequote  then (read_string (str+"\\\"") false lexbuf) else STRING (str) }
      | '\\'          { read_string str true lexbuf }
      | [^ '"' '\\']+ { read_string (str+(lexeme lexbuf)) false lexbuf }
      | eof           { raise (Exception ("String is not terminated")) }
In project directory there will be a new file ``Lexer.fs``. Include it in the project (it must be below ``Parser.fs``).

The first part of a lexer is simply F# code enclosed in ``{}``. It defines module, opens namespaces and defines helper functions. ``lexeme`` function will extract matched string from buffer. ``newline`` function updates buffer position to skip new lines characters. Notice that we open <em>Parser</em> module which contains the union type with tokens. We will use them in our productions.

Next we have a list of named regular expressions which we can use later. We can also use one regular expression within another by simply referring it by name. See line 22. ``float`` expression is referencing ``digit, frac and exp`` expressions. Space in expression means concatenation or  "and then" if you will. So for example ``frac`` expression at line 20 means: dot character then any number of digits. Characters in single quote are matched literally, for example '-' or '.'. If there are multiple expressions enclosed in ``[]`` at least one of them must be matched. You can also use ranges: ``['0'-'9'] ['a'-'z'] ['A'-'Z']``.
Meaning of repetition patterns:

* ? - 0 or 1
* \+ - 1 or more
* \* - 0 or more

After the list of regular expressions patterns we have lexing rules. Each rule contains list of patterns with productions. Same as in parser one rule can reference another.  You can also pass arguments between rules.

If you look at productions left side, it  is a pattern and right side is a F# code which will be executed when pattern is matched. F# code also must return a value (same type of value across all productions withing rule). In our case we're returning tokens, which will be later used by parser.

``read_string`` rule takes two arguments ``str ``which is accumulator, and flag ``ignorequote`` which will make rule ignore next quote. This way we can process strings which contain escaped quote like: "abc\"df".

You should be able to build project and preview generated lexer in ``Lexer.fs`` file.
5 Program
-------------

The last piece of the puzzle is to write program which will user parser. Look at below code. ``parse ``function will take text json (string) parse it and return syntax tree. You can see result in debug:

    module Program
    open FSharp.Text.Lexing
    open JsonParsing
    
    [<EntryPoint>]
    let main argv =
        let parse json = 
            let lexbuf = LexBuffer<char<.FromString json
            let res = Parser.start Lexer.read lexbuf
            res
        let simpleJson = @"{
                  ""title"": ""Cities"",
                  ""cities"": [
                    { ""name"": ""Chicago"",  ""zips"": [60601,60600] },
                    { ""name"": ""New York"", ""zips"": [10001] } 
                  ]
                }"
        let (Some parseResult) = simpleJson |> parse

    0

You can also add ``print`` function to JsonValue and show result of parsing in console:

    type JsonValue = 
      | Assoc of (string * JsonValue) list
      | Bool of bool
      | Float of float
      | Int of int
      | List of JsonValue list
      | Null
      | String of string
    //below function is not important, it simply prints values 
    static member print x = 
              match x with
              | Bool b -> sprintf "Bool(%b)" b
              | Float f -> sprintf "Float(%f)" f
              | Int d -> sprintf "Int(%d)" d
              | String s -> sprintf "String(%s)" s
              | Null ->  "Null()"
              | Assoc props ->  props 
                                 |> List.map (fun (name,value) -> sprintf "\"%s\" : %s" name (JsonValue.print(value))) 
                                 |> String.concat ","
                                 |> sprintf "Assoc(%s)"
              | List values ->  values
                                 |> List.map (fun value -> JsonValue.print(value)) 
                                 |> String.concat ","
                                 |> sprintf "List(%s)"
                                 |> sprintf "List(%s)"

And then change program to print result of parsing in console, add this line after line 18:

    printfn "%s" (JsonValue.print parseResult)