# Knot

Knot is a literate programming tool. You may wish to take a look at [other
tools][]; [nuweb][] and [noweb][] are particularly significant. There is
also [lit][], which also uses Markdown for its syntax.

Knot only tangles; it does not weave.

[other tools]: https://en.wikipedia.org/wiki/Literate_programming#Tools
[nuweb]: http://nuweb.sourceforge.net/
[noweb]: http://www.cs.tufts.edu/~nr/noweb/
[lit]: https://github.com/cdosborn/lit


## Rationale

I think literate programming is awesome, but I don't like that it's LaTeX-based
because

1. It has to go through too many transformations to get to HTML.
1. I would like to customize the HTML output more.
1. It would facilitate sharing of my documents if the literate program were
   written in a markup supported by GitHub's READMEs.

After reading [Joe Armstrong's literate program][] I decided that I could
write my own -- a strong recommendation for literate programming!

[Joe Armstrong's literate program]: https://www.sics.se/~joe/ericsson/literate/literate.html


## The Program

This is the layout of the module.

###### file:src/knot.erl
    -module(knot).
    -compile(export_all).

    ###### functions
    ###### debugging

    -ifdef(TEST).
    ###### tests
    -endif.


### Reading in Code Blocks

Unlike some other literate programming tools, we don't weave documentation
together. So the first thing we need to do is divide up an input string into
`code chunks`.

We're going to need a few utility functions for collecting the code chunks.
These `collect` functions will all return a two-tuple of the collected text and
the rest of the input, i.e. `{"foo", "bar baz buzz"}`.

#### collect_to_eol

Splits the input up until the next line break.

###### functions
    collect_to_eol(Input) ->
        collect_to_eol(Input, "").

    collect_to_eol("", Acc) ->
        {lists:reverse(Acc), ""};

    collect_to_eol([$\n | Rest], Acc) ->
        {lists:reverse(Acc), Rest};

    collect_to_eol([Char | Rest], Acc) ->
        collect_to_eol(Rest, [Char | Acc]).

###### tests
    collect_to_eol_test() ->
        {"", ""} = collect_to_eol(""),
        {"foo", "bar\nbaz"} = collect_to_eol("foo\nbar\nbaz"),
        {"foo", ""} = collect_to_eol("foo\n").


#### collect_to_fence

Splits the input up until the next line that starts with `` ``` ``.

###### functions
    collect_to_fence(Input) ->
        collect_to_fence(Input, "").

    collect_to_fence("", Acc) ->
        {lists:reverse(Acc), ""};

    collect_to_fence([$\n, $`, $`, $` | Rest], Acc) ->
        {lists:reverse(Acc), Rest};

    collect_to_fence([Char | Rest], Acc) ->
        collect_to_fence(Rest, [Char | Acc]).

###### tests
    collect_to_fence_test() ->
        {"foobar", ""} = collect_to_fence("foobar"),
        {"my\ncode\nhere", "\nmore input"} = collect_to_fence("my\ncode\nhere\n```\nmore input").


#### collect_to_unindent

Splits the input up until the next line that starts with non-white space.

###### functions
    collect_to_unindent(Input) ->
        collect_to_unindent(Input, "").

    collect_to_unindent("", Acc) ->
        {lists:reverse(Acc), ""};

    collect_to_unindent([$\n | Rest], Acc) ->
        case re:run(Rest, "^\\S") of
            {match, _} ->
                % Must put the line break back on to detect the next code block.
                {lists:reverse(Acc), [$\n | Rest]};
            nomatch ->
                collect_to_unindent(Rest, [$\n | Acc])
        end;

    collect_to_unindent([Char | Rest], Acc) ->
        collect_to_unindent(Rest, [Char | Acc]).

###### tests
    collect_to_unindent_test() ->
        {"foobar", ""} = collect_to_unindent("foobar"),
        {"    my\n    code\n    here\n", "\nmy documentation"} = collect_to_unindent("    my\n    code\n    here\n\nmy documentation").

It's important that `collect_to_unindent` doesn't consume the matched line
break. In cases where one code block immediately follows another, we need the
line break to detect the start of the next block. See `all_code` for the
pattern match that requires that.


#### collect_code

Now we can collect all the code blocks in a literate document. This function
will establish the following markup conventions.

> If you use GitHub Flavored Markdown's [fenced code blocks][], then the fence
> must start on the line immediately after the code block H6.
>
> Otherwise, the code must be indented and will end right before the first line
> that doesn't start with white space.

[fenced code blocks]: https://help.github.com/articles/github-flavored-markdown/#fenced-code-blocks

###### functions
    collect_code([$`, $`, $` | Rest]) ->
        % There might be a syntax highlighting hint that we can ignore.
        {_, Rest1} = collect_to_eol(Rest),
        collect_to_fence(Rest1);

    collect_code(Input) ->
        collect_to_unindent(Input).

###### tests
    fenced_collect_code_test() ->
        Input = "```erlang\n"
                "\n"
                "-module(foobar).\n"
                "-compile(export_all).\n"
                "\n"
                "foo() ->\n"
                "    ok.\n"
                "```\n"
                "\n"
                "documentation\n",
        Expected_block = "\n"
                         "-module(foobar).\n"
                         "-compile(export_all).\n"
                         "\n"
                         "foo() ->\n"
                         "    ok.",
        Expected_rest = "\n\ndocumentation\n",
        {Expected_block, Expected_rest} = collect_code(Input).


    indented_collect_code_test() ->
        Input = "\n"
                "    -module(foobar).\n"
                "    -compile(export_all).\n"
                "\n"
                "    foo() ->\n"
                "        ok.\n"
                "\n"
                "documentation\n",
        Expected_block = "\n"
                         "    -module(foobar).\n"
                         "    -compile(export_all).\n"
                         "\n"
                         "    foo() ->\n"
                         "        ok.\n",
        Expected_rest = "\ndocumentation\n",
        {Expected_block, Expected_rest} = collect_code(Input).


#### all_code

This will return all code blocks from the input. This establishes the markup
conventions that:

> An H6 in the leading `#` style marks a code block. For example, to output a
> file: `###### file:src/knot.erl`.


###### functions
    all_code(Input) ->
        all_code(Input, []).

    all_code("", Acc) ->
        lists:reverse(Acc);

    all_code([$\n, $#, $#, $#, $#, $#, $#, $  | Rest], Acc) ->
        {Name, Rest1} = collect_to_eol(Rest),
        {Code, Rest2} = collect_code(Rest1),
        all_code(Rest2, [{Name, Code} | Acc]);

    all_code([_ | Rest], Acc) ->
        all_code(Rest, Acc).

###### tests
    all_code_test() ->
        Input = "A sample document.\n"
                "\n"
                "\###### indented code block\n"
                "\n"
                "    Code 1, line 1.\n"
                "    Code 1, line 2.\n"
                "\n"
                "More documentation.\n"
                "\n"
                "\###### fenced code block\n"
                "```erlang\n"
                "Code 2, line 1.\n"
                "Code 2, line 2.\n"
                "```\n"
                "\n"
                "End of sample document.\n",

        Expected = [{"indented code block", "\n    Code 1, line 1.\n    Code 1, line 2.\n"},
                    {"fenced code block", "Code 2, line 1.\nCode 2, line 2."}],

        Expected = all_code(Input).

    all_code_no_intermediate_documentation_test() ->
        Input = "A sample document.\n"
                "\n"
                "\###### indented code block\n"
                "\n"
                "    Code 1, line 1.\n"
                "    Code 1, line 2.\n"
                "\n"
                "\###### another indented code block\n"
                "    Code 2, line 1.\n"
                "    Code 2, line 2.\n"
                "\n"
                "The end.\n",

        Expected = [{"indented code block", "\n    Code 1, line 1.\n    Code 1, line 2.\n"},
                    {"another indented code block", "    Code 2, line 1.\n    Code 2, line 2.\n"}],

        Expected = all_code(Input).


### Processing

Now that we've got the code blocks we can process them. We need to:

1. Unindent the code.
1. Concatenate blocks.
1. Expand macros.
1. Unescape escaped sequences.


#### Unindent Code

To remove indentation we have to first find indentation. The indentation we
need to strip is defined as the leading white space on the first line without
white space. So, given a source file like this

    Some documentation.

    ###### my code

        foo() ->
            ok.

...the indentation is four spaces because the first line with non white space
(`foo() ->`) starts with four spaces.

###### functions
    find_indentation("") ->
        "";

    find_indentation(Code) ->
        {Line, Rest} = collect_to_eol(Code),
        case re:run(Line, "^(?<white>\\s*)\\S", [{capture, [white], list}]) of
            {match, [White]} ->
                White;
            nomatch ->
                find_indentation(Rest)
        end.

###### tests
    find_indentation_test() ->
        "" = find_indentation(""),
        "" = find_indentation("    \n\t  \n    \nsomething"),
        "\t" = find_indentation("\n\n\tsomething\n\n"),
        "    " = find_indentation("\n    something").

Now we can use `find_indentation` to strip the indentation from all lines of
code.

###### functions
    unindent(Code) ->
        case find_indentation(Code) of
            "" ->
                Code;
            Indentation ->
                Pattern = [$^ | Indentation],
                unindent(Code, Pattern, [])
        end.

    unindent("", _Pattern, Lines) ->
        string:join(lists:reverse(Lines), "\n");

    unindent(Code, Pattern, Lines) ->
        {Line, Rest} = collect_to_eol(Code),
        Unindented_line = re:replace(Line, Pattern, "", [{return, list}]),
        unindent(Rest, Pattern, [Unindented_line | Lines]).

###### tests
    unindent_four_spaces_test() ->
        Input = "\n\n    foo() ->\n        ok.\n\n",
        Expected = "\n\nfoo() ->\n    ok.\n",
        Expected = unindent(Input).

    unindent_nothing_test() ->
        Input = "\nfoo() ->\n    ok.\n",
        Input = unindent(Input).

    unindent_tabs_test() ->
        Input = "\n\tfoo() ->\n\t\tok.\n",
        Expected = "\nfoo() ->\n\tok.",
        Expected = unindent(Input).

**TODO**: The `collect_to_eol` is causing the output from `unindent` to strip
the trailing line. Try and figure out if this is an issue.

And apply it to all blocks.

###### functions
    unindent_blocks(Blocks) ->
        lists:map(fun ({Name, Code}) ->
                      {Name, unindent(Code)}
                  end,
                  Blocks).

###### tests
    unindent_blocks_test() ->
        Input = [{"foo", "\tfoo() ->\n\t\tok."},
                 {"bar", "    foo() ->\n        ok."}],

        Expected = [{"foo", "foo() ->\n\tok."},
                    {"bar", "foo() ->\n    ok."}],

        Expected = unindent_blocks(Input).


#### Concatenate Blocks

When blocks of code have the same name they need to be concatenated.

###### functions
    concat_blocks(Blocks) ->
        Join_blocks = fun (Key, Acc) ->
            Values = proplists:get_all_values(Key, Blocks),
            Joined = string:join(Values, "\n"),
            [{Key, Joined} | Acc]
        end,

        lists:foldr(Join_blocks, [], proplists:get_keys(Blocks)).

###### tests
    concat_blocks_test() ->
        Input = [{"foo", "FOO"},
                 {"bar", "BAR"},
                 {"foo", "FOO"}],

        Expected = [{"foo", "FOO\nFOO"},
                    {"bar", "BAR"}],

        Expected = concat_blocks(Input).


#### Expand Macros

In other literate programming tools, the expanded macro will follow the
indentation of the line. I want to do something a little different. I want a
code block like this

    This is a list of things.
    <ul>
        <li>###### list of elements ######</li>
    </ul>

and another block like this

    one
    two
    three

to expand to

    This is a list of things.
    <ul>
        <li>one</li>
        <li>two</li>
        <li>three</li>
    </ul>

I decided I wanted to do this when I was using [noweb][] to assemble a Makefile
and wanted to assemble inline bash scripts and I had to have ` \` as a suffix
to every line. It would have been elegant to have the usage above.

I did a version of this where the macros were identified with regex. The
pattern was ugly and long. Then I wanted to use backslash to escape the macro
delimeter. It was too hard; it would only piss me off when I came back to it
later.

So, I'll start with a `collect` function that splits a line at a macro
delimeter.

This establishes a markup requirement.

> Any sequence of 6 `#` in a code block that are not to be expanded must be
> escaped with a leading backslash: `\######`.

###### functions
    collect_to_macro_delimeter(Line) ->
        collect_to_macro_delimeter(Line, "").

    collect_to_macro_delimeter("", Acc) ->
        {lists:reverse(Acc), ""};

    % Ignores escaped delimeters.
    collect_to_macro_delimeter([$\\, $#, $#, $#, $#, $#, $# | Rest], Acc) ->
        collect_to_macro_delimeter(Rest, [$#, $#, $#, $#, $#, $#, $\\ | Acc]);

    collect_to_macro_delimeter([$#, $#, $#, $#, $#, $# | Rest], Acc) ->
        {lists:reverse(Acc), Rest};

    collect_to_macro_delimeter([Char | Rest], Acc) ->
        collect_to_macro_delimeter(Rest, [Char | Acc]).

###### tests
    collect_to_macro_delimeter_test() ->
        {"foobar", ""} = collect_to_macro_delimeter("foobar"),
        {"    ", " my macro"} = collect_to_macro_delimeter("    \###### my macro"),
        {"- ", " my macro \###### -"} = collect_to_macro_delimeter("- \###### my macro \###### -"),
        {"my macro ", " -"} = collect_to_macro_delimeter("my macro \###### -"),
        {"\\\###### not a macro", ""} = collect_to_macro_delimeter("\\\###### not a macro").

And now `macro` will return `nil` or a three-tuple of `{Name, Prefix, Suffix}`
and establishes a markup convention that

> Macros with a trailing delimeter will be expanded with the line's prefix and
> suffix.

###### functions
    macro(Line) ->
        case collect_to_macro_delimeter(Line) of
            {_, ""} ->
                % No macro in this line.
                nil;

            {Prefix, Rest} ->
                % Rest contains the macro name and, potentially, another
                % delimeter before the suffix.
                {Padded_name, Suffix} = collect_to_macro_delimeter(Rest),
                {string:strip(Padded_name), Prefix, Suffix}
        end.

###### tests
    macro_test() ->
        nil = macro("foobar"),
        {"my macro", "    ", ""} = macro("    \###### my macro"),
        {"my macro", "    <li>", "</li>"} = macro("    <li>\###### my macro \######</li>").


With `expand_macros`, a source file like this

    ###### my code
        ###### things
        - ###### things ###### -

    ###### things
        one
        two

will expand to

    one
    two
    - one -
    - two -

###### functions
    expand_macros(Code, Blocks) ->
        expand_macros(Code, Blocks, []).

    expand_macros("", _Blocks, Acc) ->
        string:join(lists:reverse(Acc), "\n");

    expand_macros(Code, Blocks, Acc) ->
        {Line, Rest} = collect_to_eol(Code),
        case macro(Line) of
            nil ->
                expand_macros(Rest, Blocks, [Line | Acc]);

            {Name, Prefix, Suffix} ->
                case proplists:get_value(Name, Blocks) of
                    undefined ->
                        io:format("Warning: code block named ~p not found.~n", [Name]),
                        expand_macros(Rest, Blocks, [Line | Acc]);

                    Code_to_insert ->
                        New_lines = re:split(Code_to_insert, "\n", [{return, list}]),
                        Wrapped = lists:map(fun (X) -> Prefix ++ X ++ Suffix end, New_lines),
                        expand_macros(Rest, Blocks, [string:join(Wrapped, "\n") | Acc])
                end
        end.

###### tests
    expand_macros_test() ->
        Input_code = "\n"
                     "start\n"
                     "\###### things\n"
                     "- \###### things \###### -\n"
                     "end\n",
        Input_blocks = [{"things", "one\ntwo"}],
        Expected = "\nstart\none\ntwo\n- one -\n- two -\nend",
        Expected = expand_macros(Input_code, Input_blocks).


And now we have to do that for every block.

###### functions
    expand_all_macros(Blocks) ->
        expand_all_macros(Blocks, Blocks, []).

    expand_all_macros([], _Blocks, Acc) ->
        lists:reverse(Acc);

    expand_all_macros([{Name, Code} | Rest], Blocks, Acc) ->
        expand_all_macros(Rest, Blocks, [{Name, expand_macros(Code, Blocks)} | Acc]).


###### tests
    expand_all_macros_test() ->
        Input = [{"first one", "First.\n\###### list of things"},
                 {"second one", "This...\n-\###### list of things \######-\nis the second."},
                 {"All the things!", "\###### first one \######\n* \###### second one \###### *\nDone."},
                 {"list of things", "one\ntwo"}],

        Expected = [{"first one", "First.\none\ntwo"},
                    {"second one", "This...\n-one-\n-two-\nis the second."},
                    {"All the things!", "First.\none\ntwo\n* This... *\n* -one- *\n* -two- *\n* is the second. *\nDone."},
                    {"list of things", "one\ntwo"}],

        Expected = expand_all_macros(expand_all_macros(Input)).


#### Unescape Escaped Sequences

If you're in a code block and need to use `######`, then you must escape it. As
a final step in the processing of literate files, I will unescape the escaped
H6s.

The backslash is used in regex patterns, so we need to double up.

###### functions
    unescape(Code) ->
        re:replace(Code, "\\\\\######", "\######", [global, {return, list}]).

###### tests
    unescape_test() ->
        "foo\n    \###### not a macro\nbar" = unescape("foo\n    \\\###### not a macro\nbar"),
        "- \\\###### really not a macro \\\###### -" = unescape("- \\\\\###### really not a macro \\\\\###### -"),
        "\###### h6 of another Markdown document \######" = unescape("\\\###### h6 of another Markdown document \\\######").

And do it for all blocks.

###### functions
    unescape_blocks(Blocks) ->
        unescape_blocks(Blocks, []).

    unescape_blocks([], Acc) ->
        lists:reverse(Acc);

    unescape_blocks([{Name, Code} | Rest], Acc) ->
        unescape_blocks(Rest, [{Name, unescape(Code)} | Acc]).

###### tests
    unescape_blocks_test() ->
        Input = [{"foo", "\\\######"},
                 {"bar", "bar"},
                 {"baz", "\\\###### h6 of another Markdown document \\\######"}],

        Expected = [{"foo", "\######"},
                    {"bar", "bar"},
                    {"baz", "\###### h6 of another Markdown document \######"}],

        Expected = unescape_blocks(Input).

### Debugging

I'll want to provide something on the command line to help debug each stage in
the file processing.

The good thing about the functions I've already written is that they only take
one argument for input, so it's pretty easy to pass in a string of text
representing a partial document. However, I'll want to pass in a source file
and just see what code blocks are detected.

First I'll need to read files in and output blocks of code in various stages of
processing.

###### debugging
    read_file(File_name) ->
        {ok, Binary} = file:read_file(File_name),
        binary_to_list(Binary).

    print_blocks(Blocks) ->
        lists:foreach(fun ({Name, Code}) ->
                          io:format("~s~n-----~n~s~n-----~n~n",
                                    [Name, Code])
                      end,
                      Blocks).

And now we can print the various stages.

###### debugging
    print_code(File_name) ->
        print_blocks(
            all_code(
                read_file(File_name))).

    print_unindented_code(File_name) ->
        print_blocks(
            unindent_blocks(
                all_code(
                    read_file(File_name)))).

    print_concatenated_code(File_name) ->
        print_blocks(
            concat_blocks(
                unindent_blocks(
                    all_code(
                        read_file(File_name))))).

    print_expanded_code(File_name) ->
        print_blocks(
            expand_all_macros(
                concat_blocks(
                    unindent_blocks(
                        all_code(
                            read_file(File_name)))))).

    print_unescaped_code(File_name) ->
        print_blocks(
            unescape_blocks(
                expand_all_macros(
                    concat_blocks(
                        unindent_blocks(
                            all_code(
                                read_file(File_name))))))).


### Writing Files

The input is fully processed now. This program wouldn't be useful without we
write some files. This is pretty simple because we only need to find the blocks
with names prefixed with `file:`.

###### functions
    file_blocks(Blocks) ->
        file_blocks(Blocks, []).

    file_blocks([], Acc) ->
        lists:reverse(Acc);

    file_blocks([{[$f, $i, $l, $e, $: | _] = Name, Code} | Rest], Acc) ->
        file_blocks(Rest, [{Name, Code} | Acc]);

    file_blocks([_ | Rest], Acc) ->
        file_blocks(Rest, Acc).

###### tests
    file_blocks_test() ->
        Input = [{"file:a", "a"},
                 {"not a file", "not a file"},
                 {"file:b", "b"}],
        Expected = [{"file:a", "a"},
                    {"file:b", "b"}],
        Expected = file_blocks(Input).

Might want to debug it, too.

###### debugging
    print_file_blocks(File_name) ->
        print_blocks(
            file_blocks(
                unescape_blocks(
                    expand_all_macros(
                        concat_blocks(
                            unindent_blocks(
                                all_code(
                                    read_file(File_name)))))))).

Given file blocks, write them. The reason we're using a `Base_directory` for
these functions is that the output files will be relative to the source file.
This gives us another markup requirement:

> File names given in `file:` blocks are relative to the source file.

###### functions
    file_name(Base_directory, File_name) ->
        filename:nativename(filename:absname_join(Base_directory, File_name)).

    write_file(Base_directory, File_name, Contents) ->
        Fn = file_name(Base_directory, File_name),
        ok = file:write_file(Fn, Contents),
        Fn.

###### tests
    file_name_test() ->
        "test_files/foobar.txt" = file_name("test_files", "foobar.txt"),
        "/path/to/repository/src/knot.erl" = file_name("/path/to/repository", "src/knot.erl").

    write_file_test() ->
        "test_files/test.txt" = write_file("test_files", "test.txt", "write_file_test\n"),
        {ok, <<"write_file_test\n">>} = file:read_file(file_name("test_files", "test.txt")),
        file:delete(file_name("test_files", "test.txt")).


### Putting it All Together

`process_file` will do everything!

###### functions
    process_file(File_name) ->
        Base_directory = filename:dirname(File_name),
        Files = file_blocks(
                    unescape_blocks(
                        expand_all_macros(
                            concat_blocks(
                                unindent_blocks(
                                    all_code(
                                        read_file(File_name))))))),
        write_file_blocks(Base_directory, Files).

    write_file_blocks(_Base_directory, []) ->
        ok;

    write_file_blocks(Base_directory, [{[$f, $i, $l, $e, $: | File_name], Contents} | Rest]) ->
        write_file(Base_directory, File_name, Contents),
        write_file_blocks(Base_directory, Rest).

###### tests
    process_file_test() ->
        ok = process_file("test_files/process_file_test.md"),
        Expected = read_file("test_files/process_file_test.js.expected_output"),
        Actual = read_file("test_files/process_file_test.js") ++ "\n",
        io:format("~p~n~p~n", [Expected, Actual]),
        Expected = Actual,
        file:delete("test_files/process_file_test.js").

**TODO**: The process_file test has to manually append a new line because my
editor always appends a new line at the end of the file. I had to add it to
pass the test -- I think this is because of the way I use `collect_to_eol`
without returning the matched line break. But is this an issue? I'm not sure.

Handle multiple files.

###### functions
    process_files([]) ->
        ok;

    process_files([File | Files]) ->
        process_file(File),
        process_files(Files).