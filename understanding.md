# Understanding

## Architecture
At the top we have the main package with 2 main package APIs that it exposes.
As far as i understand these should be the only two methods exposed, which it
is not.

It is LoadSchema and LoadQuery, these functions has 2 deriviations, that
adds a panic but doesn't matter.

To call these functions, they take in a arbitrary list of the type *ast.Source
. Which is also a part of the package. I think this is kind of dirty, 
i think that the API should take in whatever parameters the ast.Source
consists of.

Especially if you look at the `architecture.png`, i think that it is weird
that all of the pakages rely on ast, it should preferably be a hierarchy of
some sort. This kind of implementation gives a higher coupling, and makes
it harder to understand how to use the software and how everything is coupled.
If you have to read another subpackage documentation just to use the main
package API you're already making it harder.

The source type looks like so:

type Source struct {
	Name    string
	Input   string
	BuiltIn bool
}

Where the `Name` is the file name, `Input` is the actual file contents.
Looks like the content is regular graphql specification stuff like where
newlines is shown with \n, comments has # before them etc. 
There is also a `BuiltIn` variable, which covers whether the Source is part
of the official specification and not a custom *.graphql file. E.g. the prelude
that is being loaded everytime, as the first part before the rest of the
*.graphql files is loaded as sources.

https://github.com/vektah/gqlparser/blob/master/validator/prelude.graphql

So when all of these sources is collected, we have a rather funny line which
unpacks all of the sources, just to append them to the prelude.
It is then given in to the validator.LoadSchema, which takes in this
[]*ast.Source and returns a (*ast.Schema, error).

When all of the sources has been parsed in, they're going to be parsed indivi-
dually and then merged into the ast.SchemaDocument one-by-one.

The SchemaDocument is basically the entire AST as far as i understand.

Every single source when it gets parsed, we create ast.parser which
needs a lexer, and the lexer takes in the source.
When this object is created we can call `parseSchemaDocument` on it.

This function looks rather important. It starts out by peeking
the position

At the `lexer` level of the implementation, when we have to peek at the next
character. It is important to notice that we have one lexer pr. source,
so when the lexer calls some function and uses local variables it doesn't
affect other sources than the one being parsed.
We call a `ReadToken` which calls ws(), probably an abreviation for whitespace.
The name `ws` should probably be changed though, and it's comment is bad as
well. The comment talks about moving the end to include all of the whitespace,
but it also ends up including all these chars: '\t', '\n', '\r', ',' and then
whitespace also ofc.
When it reaches the first char which is none of these, it returns.
The ReadToken then sets the end of all these characters, to the start of the
same lexer. Meaning that for some time lexer.start = lexer.end. It then
uses the len(lexer.Input) and lexer.end to check if we're at a EOF which
it would then create and return(i guess the token cannot be read from the
input).
Then we actually fetch out the character located at lexer.Input[s.start],
and compare this character with a switch case to find out which token we
should return.
A lot of the cases returns a TokenValue, but with the Value set to a empty
string weirdly enough. I guess it comes in handy at some point.
If the character it peeks is more complicated, like a '#' starting a comment.
We call another function that looks to be reading the comment until the end,
and then returns the comment as a token, this token it returns is not being
used though. We simply call *lexer.ReadToken() again recursively, which
i guess jumps to the next token after the comment.
If we get a [a-zA-Z _] we will read a name, and return a name token.
If we get a '-' or [0-9] we will return a number token.
Getting the new char "  will try to read a block comment which is just
multi line comments done with triple quotations like:

"""
herpderp
"""

The way the overall `parseSchemaDocument` method works, is that it calls this
peek method on the lexer in a for loop until it hits a EOF token, 
while running it initially checks for the things
that could actually exist in the outer scope of the *.graphql file.
These are considered the tokens comment/blockstring/name, if it is not either
of these, it throws a error. 
If it is the comment, it continues to the next token.
If it is a name, it tries to understand what token value e.g. `interface`,
`directive`, `type`, `scalar` etc. it will handle this token in it's 
entirety regardless of nesting. And when it returns it will continue to the
next token in the outer scope.



All of the `definitions` and `extensions` inherits the parent ast.Source
`BuiltIn` value before the Schema gets returned

