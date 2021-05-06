
<div style="height: 88vh;">

# Clojure Document

<div style="height: 84%; overflow-y: auto">


* Full featured text document model
* Clojure Lexer
* Clojure LISP Token Cursor
* Integrates into VS Code as a two way mirror of the VS Code Editors
* Powers Paredit, rainbow parens, ”The Current Form”, etcetera, etcetera


## Lexer

* Tokenizes all the things
* Line based
* Regex party

```ts
// open parens
toplevel.terminal("open", /((?<=(^|[\(\)\[\]\{\}\s,]))['`~#@?^]\s*)*['`~#@?^]*[\(\[\{"]/, (l, m) => ({ type: "open" }))

// close parens
toplevel.terminal("close", /\)|\]|\}/, (l, m) => ({ type: "close" }))

// ignores
toplevel.terminal("ignore", /#_/, (l, m) => ({ type: "ignore" }))

// symbols, allows quite a lot, but can't start with `#_`, anything numeric, or a selection of chars
toplevel.terminal("id",
  /(['`~#^@]\s*)*(((?<!#)_|[+-](?!\d)|[^-+\d_()[\]\{\}#,~@'`^\"\s:;\\])[^()[\]\{\},~@`^\"\s;\\]*)/,
  (l, m) => ({ type: "id" }))
```


## Token Cursor

* Structural LISP cursor
* Dates back to (possibly before) Zmacs on LISP Machines
* Our cursor ”knows” about Clojure

```ts
    /**
     * Moves this token forward one s-expression at this level.
     * If the next non whitespace token is an open paren, skips past it's matching
     * close paren.
     *
     * If the next token is a form of closing paren, does not move.
     *
     * @returns true if the cursor was moved, false otherwise.
     */
    forwardSexp(skipComments = true, skipMetadata = false, skipIgnoredForms = false): boolean {
        // TODO: Consider using a proper bracket stack
        const stack = [];
        let isMetadata = false;
        this.forwardWhitespace(skipComments);
        if (this.getToken().type === 'close') {
            return false;
        }
        if (this.tokenBeginsMetadata()) {
            isMetadata = true;
        }
        while (!this.atEnd()) {
            this.forwardWhitespace(skipComments);
            const token = this.getToken();
            switch(token.type) {
                case 'comment':
                    this.next();
                    this.next();
                    break;
                case 'prompt':
                    this.next();
                    this.next();
                    break;
                case 'ignore':
                    if (skipIgnoredForms) {
                        this.next();
                        this.forwardSexp(skipComments, skipMetadata, skipIgnoredForms);
                        break;
                    }
                case 'id':
                case 'lit':
                case 'kw':
                case 'junk':
                case 'str-inside':
                    if (skipMetadata && this.getToken().raw.startsWith('^')) {
                        this.next();
                    } else {
                        this.next();
                        if (stack.length <= 0) {
                            return true;
                        }
                    }
                    break;
                case 'close':
                    const close = token.raw;
                    let open: string;
                    while (open = stack.pop()) {
                        if (validPair(open, close)) {
                            this.next();
                            break;
                        }
                    }
                    if (skipMetadata && isMetadata) {
                        this.forwardSexp(skipComments, skipMetadata);
                    }
                    if (stack.length <= 0) {
                        return true;
                    }
                    break;
                case 'open':
                    stack.push(token.raw);
                    this.next();
                    break;
                default:
                    this.next();
                    break;
            }
        }
    }
```

* Growing test bed

```ts
describe('Token Cursor', () => {
    describe('forwardSexp', () => {
        it('moves from beginning to end of symbol', () => {
            const a = docFromTextNotation('(a(b(|c•#f•(#b •[:f :b :z])•#z•1)))')
            const b = docFromTextNotation('(a(b(c|•#f•(#b •[:f :b :z])•#z•1)))')
            const cursor: LispTokenCursor = a.getTokenCursor(a.selectionLeft);
            cursor.forwardSexp();
            expect(cursor.offsetStart).toBe(b.selectionLeft);
        });
        it('moves from beginning to end of nested list ', () => {
            const a = docFromTextNotation('|(a(b(c•#f•(#b •[:f :b :z])•#z•1)))')
            const b = docFromTextNotation('(a(b(c•#f•(#b •[:f :b :z])•#z•1)))|')
            const cursor: LispTokenCursor = a.getTokenCursor(a.selectionLeft);
            cursor.forwardSexp();
            expect(cursor.offsetStart).toBe(b.selectionLeft);
        });
        ,,,
    ,,,
,,,  
```

* Powers Paredit (and many more Calva things)

```ts
export function raiseSexp(doc: EditableDocument, start = doc.selectionLeft, end = doc.selectionRight) {
    const cursor = doc.getTokenCursor(end);
    const [formStart, formEnd] = cursor.rangeForCurrentForm(start);
    const isCaretTrailing = formEnd - start < start - formStart;
    const startCursor = doc.getTokenCursor(formStart);
    let endCursor = startCursor.clone();
    if (endCursor.forwardSexp()) {
        let raised = doc.model.getText(startCursor.offsetStart, endCursor.offsetStart);
        startCursor.backwardList();
        endCursor.forwardList();
        if (startCursor.getPrevToken().type == "open") {
            startCursor.previous();
            if (endCursor.getToken().type == "close") {
                doc.model.edit([
                    new ModelEdit('changeRange', [startCursor.offsetStart, endCursor.offsetEnd, raised])
                ], {
                    selection: new ModelEditSelection(isCaretTrailing ?
                        startCursor.offsetStart + raised.length :
                        startCursor.offsetStart)
                });
            }
        }
    }
}
```


</div>

</div>


---

[Start](hello.md) > [VS Code](vscode.md) > [Calva](calva.md) > [Tao of Calva](tao-of-calva.md) > [Development](calva-dev.md) > [Clojure Document](clojure-document.md) > [AMA](ama.md)