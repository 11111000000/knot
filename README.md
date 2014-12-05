# Knot

Knot is a literate programming tool. You may wish to take a look at [other
tools][]; [nuweb][] and [noweb][] are particularly significant. There is
also [lit][], which also uses Markdown for its syntax.

[other tools]: https://en.wikipedia.org/wiki/Literate_programming#Tools
[nuweb]: http://nuweb.sourceforge.net/
[noweb]: http://www.cs.tufts.edu/~nr/noweb/
[lit]: https://github.com/cdosborn/lit


## Rationale

I think literate programming is awesome, but I don't like that it's
LaTeX-based because

1. It has to go through too many transformations to get to HTML.
1. I would like to customize the HTML output more.
1. It would facilitate sharing of my documents if the literate program were
   written in a markup supported by GitHub's READMEs.

After reading [Joe Armstrong's literate program][] I decided that I could
write my own -- a strong recommendation for literate programming!

[Joe Armstrong's literate program]: https://www.sics.se/~joe/ericsson/literate/literate.html


## Usage

Knot files are written in Markdown.

They will use the `######`-style H6 to denote potential code blocks. If
the name of the block starts with `file:` then a file with that name will
be written (relative to the source document).

Two types of code blocks are supported: indented code blocks (typical of
Markdown) and GitHub Flavored Markdown's fenced code blocks.

Code expansion also uses `######`. As with other literate programming
tools, indentation is maintained. However, there is a feature that I
believe is unique to knot that will expand code with the line's prefix and
suffix.

If you need to escape the `######` (as I have, since Knot is
self-hosting), prefix it with a backslash, e.g. `\######`.

### Example

A file like this:

    This is a simple literate program that outputs `my_file.txt`.

    ###### file:my_file.txt
        I am in my file.

        Some things:

        - ###### my things ###### -

        ###### footer

    My things are just three numbers.

    ###### my things
        one
        two
        three

    And the footer just shows the abbreviated style.

    ###### footer
        It tasted like a foot.

...will output `my_file.txt` like this:

    I am in my file.

    Some things:

    - one -
    - two -
    - three -

    It tasted like a foot.

## Installation

Install Erlang.

    sudo apt-get install erlang

Put the [knot](knot) escript somewhere in your path.