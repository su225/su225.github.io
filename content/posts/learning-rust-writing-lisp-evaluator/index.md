---
title: "Learning Lifetime parameters in Rust: Writing a simple Lisp evaluator"
date: 2022-08-24T10:48:17+05:30
tags: [leetcode,rust,learning]
---

[Link to the problem description](https://leetcode.com/problems/parse-lisp-expression/)

**Brief description** - We are given a valid Lisp expression. The expression types are
1. **Integer**: An integer which evaluates to the same value (positive or negative)
2. **Let**: Used to introduce new variables into the scope
3. **Add**: `(add e1 e2)` where `e1` and `e2` expressions evaluate to integers. It evaluates to the sum of the two
4. **Mult**: `(mult e1 e2)` where `e1` and `e2` expressions evaluate to integers. It evaluates to the product of the two

The problem is to parse, evaluate and return the resulting integer. For instance
```plaintext
(let x 2 
  (mult x (let x 3 y 4 
            (add x y))))
```
evaluates to `14` as follows
```plaintext
  (let x 2 (mult x (let x 3 y 4 (add x y))))
= (let x 2 (mult x (let x 3 y 4 (add 3 4)))) -- substitute x=3, y=4
= (let x 2 (mult x 7))                       -- evaluate (add 3 4)
= (let x 2 (mult 2 7))                       -- substitute x=2
= 14                                         -- evaluate (mult 2 7)
```

The solution consists of writing three parts
1. **Lexer** - tokenizes the input stream
2. **Evaluator** - builds the [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) on the fly, walks the tree, maintains `Environment` containing the value bound for each variable and evaluates the expression. As shown in the example above, the innermost value is considered.

As one of my main aims was to learn the concepts of borrow and lifetimes, I wrote it to minimize copying as much as possible and keep references to parts of the input stream instead, especially for identifiers. Hence, you will find lifetime specifiers. Please note that zero-copy does not necessarily mean zero-allocations. 

{{< toc >}}

## Lexer
Lexer tokenizes the input stream. That is, it classifies different parts of the text according to functions - like numbers, punctuation (open and close parentheses), identifier etc. Lexer turns a stream of bytes (`u8`) into a stream of `Token` defined as follows
```rust
#[derive(Debug, Eq, PartialEq)]
enum Token<'a> {
    OpenParentheses,
    CloseParentheses,
    Identifier(&'a str),
    Number(i32),
}

struct TokenStream<'b> {
    offset: usize,
    s: &'b [u8],
}

impl<'b> Iterator for TokenStream<'b> {
    type Item = Token<'b>;
    fn next(&mut self) -> Option<Self::Item> {
        // ... lexer implementation
    }
}
```
Here the lifetime parameter `'b` on `Token<'b>` with respect to `TokenStream<'b>` can be read as **"The references in `Token` struct returned by the iterator is valid as long as the underlying reference to `s` in `TokenStream` which is a bunch of bytes is also valid"**. This byte slice is the input expression. No copying is necessary. In other words, the lifetime parameters say that the references held by the structs are valid as long as the bytes representing the underlying bytes is also valid.

While reading lifetime parameters, it is helpful to read it with respect to the scope from where references are used (like passing parameters etc). One of the mistakes I used to do was to try to interpret it standalone which didn't make sense at that time.

### Initializing `TokenStream` state
```rust
impl<'b> TokenStream<'b> {
    fn from_string<'a: 'b>(s: &'a String) -> TokenStream<'b> {
        TokenStream {
            offset: 0,
            s: s.as_bytes(),
        }
    }
    // .. rest of the code
}
```
The interesting part for someone learning not to fight the borrow checker is this part
```rust
fn from_string<'a: 'b>(s: &'a String) -> TokenStream<'b>
```
There are two lifetime parameters `'a` and `'b` with constraints defined on them. `s` here is the input String. The constraint `'a: 'b` says that lifetime `'a` is at least as long as lifetime `'b`. Here, `'a` is the lifetime of the input string and `'b` is the lifetime of the `TokenStream` iterator built on top of that. The iterator does not consume the data in the sense that it cannot own it. Another reminder - the lifetime parameters, by themselves, are not that useful and would be a nuisance. They only make sense in cases of comparing lifetimes. Here, we are comparing the lifetimes of the input and the stream holding reference to it. 

So the iterator built on top of the immutably borrowed underlying input byte stream should **NOT** live longer than the input itself as this causes **dangling reference** compromising memory safety properties provided by safe Rust. This should cause an error.

### Tokenizing identifier and numbers
The identifier would be a `&str` which is a reference to the underlying input bytes. Here we assume ASCII and hence all characters would be single byte. 

We advance each byte as long as we find a character that could be part of an identifier (including reserved words like `let`, `add`, `mult`) until we find something else. In this case, it is either a space or open/close parentheses. We note start and end offsets of the identifiers and get **reference** to that part of the input stream. **There is no string copying**. 

> **NOTE**: Internally `from_utf8` validates if the bytes are UTF-8 encoded and ultimately calls `mem::transmute` which **re-interprets** those bytes as a different type (very unsafe). As we assume valid expression and ASCII only (which is also valid UTF-8), we can skip the check by calling `from_utf8_unchecked` which is **unsafe**. 
```rust
impl<'b> TokenStream<'b> {
    // .. other code

    fn parse_identifier(&mut self) -> &'b str {
        let start_offset = self.offset;
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];

            // We don't check for "starting with digit"
            // case as it is already checked by next()
            // before entering this method.
            if cur_byte == b'(' || cur_byte == b')' || cur_byte == b' ' {
                break;
            }
            self.offset += 1;
        }
        assert!(start_offset < self.offset);
        std::str::from_utf8(&self.s[start_offset..self.offset])
            .expect("identifier to be correctly parsed")
    }

    // .. rest of code
}
```
Tokenizing numbers is much simpler (Maybe there is an idiomatic way of writing this. But this is enough for our purposes). Writing this for completeness.
```rust
impl<'b> TokenStream<'b> {
    // .. other code

    fn parse_number(&mut self) -> i32 {
        let mut val = 0;
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];
            if cur_byte < b'0' || cur_byte > b'9' {
                break;
            }
            let cur_digit = (cur_byte - b'0') as i32;
            val = val * 10 + cur_digit;
            self.offset += 1;
        }
        val
    }

    // .. rest of code
}
```

### Implementing `Iterator` trait for `TokenStream`
We can go through the stream at once and eagerly tokenize it into `Vec<Token>`. However, it requires a lot of wasteful allocation. We only need a few tokens to make parsing decisions. So, tokenization is performed lazily. This is implemented by an iterator. Implementing `Iterator` trait requires implementing `next()` function which **may** return the next token in the stream.

We tokenize on the fly from the current input byte offset and return it to the caller as shown below. Notice that there are **no extra heap allocations** (the data part of `String` is on the heap though. Both the `TokenStream` and `Token::Identifier(&str)` reference that piece of memory directly)
```rust
impl<'b> Iterator for TokenStream<'b> {
    type Item = Token<'b>;

    fn next(&mut self) -> Option<Self::Item> {
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];
            match cur_byte {
                b'(' => {
                    self.offset += 1;
                    return Some(Token::OpenParentheses);
                }
                b')' => {
                    self.offset += 1;
                    return Some(Token::CloseParentheses);
                }
                b'-' => {
                    self.offset += 1;
                    let num = self.parse_number();
                    return Some(Token::Number(-num));
                }
                b'0'..=b'9' => {
                    let num = self.parse_number();
                    return Some(Token::Number(num));
                }
                b'a'..=b'z' => {
                    let id = self.parse_identifier();
                    return Some(Token::Identifier(id));
                }
                b' ' => {
                    self.offset += 1;
                    continue;
                }
                ch => panic!("unexpected character: {}", ch as char),
            }
        }
        None
    }

}
```

## Expression evaluation
After tokenizing the input stream, it has to be parsed to construct the Abstract Syntax Tree and evaluate. Luckily, Lisp syntax is easy to parse. In this solution, the Abstract Syntax Tree is not constructed explicitly. We take the token stream, parse on the fly, maintain the "environment" consisting of binding from an identifier to its value and evaluate on the fly.

The grammar to be parsed is as follows
```plaintext
<expr> -> (add <expr> <expr>)
        | (mult <expr> <expr>)
        | (let [<ident> <expr>]+ <expr>)
        | <number>
        | <ident>
```
The outline of the evaluator.
```rust
struct Environment<'a> {
    sym_vals: Vec<HashMap<&'a str, i32>>,
}

struct LispEvaluator<'b, 'a: 'b> {
    s: std::iter::Peekable<TokenStream<'a>>,
    env: Environment<'b>,
}

impl<'b, 'a> LispEvaluator<'b, 'a>
where
    'a: 'b,
{
    fn evaluate(&mut self) -> Result<i32, EvalError>;
}
```
The lifetime variables are interesting here and need some explanation. The `Environment` needs a lifetime specifier because of the key which is a reference `&'a str`. The bound `'a : 'b` says that "`'a` lives at least as long as `'b`" or in other words, `'b` does not outlive `'a`. 

* `'a` is the lifetime of the underlying input byte stream (remember `s: &'b [u8]`?).
* `'b` is the lifetime of the identifier key in the hash table wrapped by `Environment`. 

`'b` does NOT outlive `'a` would mean that the reference `&str` added to the HashMap in `Environment` does not outlive the underlying input byte stream. It makes sense as allowing it would cause dangling reference.

### Environment
Environment provides the context for evaluation. It keeps track of the value of bindings, implements shadowing requirements - pick the value of the innermost scope in case the identifier is declared more than once. 

* Every `let` expression creates a scope and when its evaluation is complete, it is destroyed.
* Every binding added through the `let` expression is added to the innermost scope (let us call it the current scope - the scope that was recently created at the start of evaluating `let`).

The operations are straightforward and the implementation is not provided in this section
```rust
impl<'a> Environment<'a> {
    fn new() -> Self;
    fn add_binding(&mut self, sym: &'a str, val: i32);
    fn lookup_symbol(&self, sym: &'a str) -> Option<i32>;
    fn create_scope(&mut self);
    fn destroy_scope(&mut self);
}
```
The lifetime parameters on methods `add_binding` and `lookup_symbol` need some explanation. `'a` here specifies the constraint that the `sym` lives at least as long as the structure itself (or the Environment structure does not outlive the passed-in reference).

### Expression evaluation
As mentioned earlier, the token is obtained from `TokenStream` which **lazily** tokenizes the bytes and does NOT make additional copies of the identifier (It is all `&str`). Parsing is done in a simple top-down manner. The implementation is straightforward. It maps neatly to the grammar specified earlier.
```rust
const ADD_FUNC: &'static str = "add";
const MULTIPLY_FUNC: &'static str = "mult";
const LET_KEYWORD: &'static str = "let";

impl<'b, 'a> LispEvaluator<'b, 'a>
where
    'a: 'b,
{
    // .. rest of code
    
    fn evaluate(&mut self) -> Result<i32, EvalError> {
        self.evaluate_expr()
    }

    fn evaluate_expr(&mut self) -> Result<i32, EvalError> {
        let first_tok = self.s.next();
        match first_tok {
            Some(Token::OpenParentheses) => {
                let second_tok = self.s.next();
                let val = match second_tok {
                    Some(Token::Identifier(ADD_FUNC)) => self.evaluate_addition(),
                    Some(Token::Identifier(MULTIPLY_FUNC)) => self.evaluate_multiplication(),
                    Some(Token::Identifier(LET_KEYWORD)) => self.evaluate_let_block(),
                    Some(tok) => Err(EvalError(format!(
                        "unknown token: {:?}, expected add,mult,let",
                        tok
                    ))),
                    None => Err(EvalError(format!("expected add,mult,let, but got nothing"))),
                };
                self.ensure_closing_parentheses().and(val)
            }
            Some(Token::Identifier(ident)) => self.evaluate_identifier(ident),
            Some(Token::Number(n)) => Ok(n),
            Some(tok) => Err(EvalError(format!("unexpected token: {:?}", tok))),
            None => Err(EvalError(format!(
                "no token found for evaluating expression"
            ))),
        }
    }

    // .. rest of code
}
```

#### Evaluating `add` and `mult` expressions
The `evaluate_*` are a set of mutually recursive functions. The functions `evaluate_addition` and `evaluate_multiplication` are straightforward. They parse the expressions representing operands and perform respective operations.
```rust
impl<'b, 'a> LispEvaluator<'b, 'a>
where
    'a: 'b,
{
    // .. rest of code

    fn evaluate_addition(&mut self) -> Result<i32, EvalError> {
        let left_val = self.evaluate()?;
        let right_val = self.evaluate()?;
        Ok(left_val + right_val)
    }

    fn evaluate_multiplication(&mut self) -> Result<i32, EvalError> {
        let left_val = self.evaluate()?;
        let right_val = self.evaluate()?;
        Ok(left_val * right_val)
    }

    // .. rest of code
}
```

#### Evaluating the `let` expression
This is a bit more complex than `add` and `mult` expressions. Hence, the code is annotated with many comments. 
```rust
impl<'b, 'a> LispEvaluator<'b, 'a>
where
    'a: 'b,
{
    // .. rest of code

    fn evaluate_let_block(&mut self) -> Result<i32, EvalError> {
        // The first one would be an identifier and the second
        // one would be an expression. The last one would be an
        // expression followed by closing parentheses. An identifier
        // is also a valid expression and hence that works.
        self.env.create_scope();
        let expr_result;
        loop {
            let ident_tok = self.s.peek();
            match ident_tok {
                Some(&Token::Identifier(ident)) => {
                    self.s.next();
                    let next_tok = self.s.peek();

                    // If it is an identifier followed by closing parentheses
                    // then this should be treated as an expression and should
                    // be evaluated as such. If it not a closing parentheses,
                    // then it should be evaluated as an expression and should
                    // be added to the environment.
                    if next_tok == Some(&Token::CloseParentheses) {
                        expr_result = self.evaluate_identifier(ident);
                        break;
                    } else {
                        let expr_val = self.evaluate_expr()?;
                        self.env.add_binding(ident, expr_val);
                    }
                }
                Some(_) => {
                    // If there is something else in a position where identifier
                    // is expected, then this must be an expression evaluated in
                    // the context of all the let bindings seen so far.
                    expr_result = self.evaluate_expr();
                    break;
                }
                None => {
                    expr_result = Err(EvalError(format!(
                        "expected expression at the end of let block"
                    )));
                    break;
                }
            };
        }
        self.env.destroy_scope();
        expr_result
    }

    fn evaluate_identifier(&mut self, ident: &'a str) -> Result<i32, EvalError> {
        self.env
            .lookup_symbol(ident)
            .ok_or(EvalError(format!("cannot lookup symbol: {}", ident)))
    }

    // .. rest of code
}
```
I'll leave out the parsing logic as it is not that interesting. Notice that creation (`create_scope`) and deletion of scope (`delete_scope`). The `add_binding` makes sure that if the identifier is already defined in outer scopes (the ones before `create_scope` was called), then it is shadowed. The `destroy_scope` method removes this shadowing to restore older values. Hence the following expression correctly evaluates to `18`, not `20` or `16`
```plaintext
(let x 10 (add (let x 8 x) x))
```
How does it work?
1. When the `let` is encountered an empty scope is created with `create_scope`. Let me represent it as `[]` where the rightmost is the innermost scope.
1. When the `x 10` is evaluated, the scope is updated to `[x->10]`
1. When the `let` part of first operand of `add` is seen, the scopes look like `[x->10][]`
1. Now, `x 8` updates the bindings as `[x->10][x->8]`
1. To evaluate `x`, `evaluate_identifier` looks at the **innermost** scope and returns `8`.
1. When we are done evaluating the inner `let` expression, the `destroy_scope` destroys the innermost scope and all the bindings that are added to it. So, it looks like `[x->10]`.
1. Now, the second operand `x` is just an identifier. The value returned by `lookup_symbol` would be `10`
1. Finally `(add 8 10)` is evaluated to `18`.

## Full implementation
{{< highlight rust >}}
use std::collections::HashMap;

#[derive(Debug, Eq, PartialEq)]
enum Token<'a> {
    OpenParentheses,
    CloseParentheses,
    Identifier(&'a str),
    Number(i32),
}

struct TokenStream<'b> {
    offset: usize,
    s: &'b [u8],
}

impl<'b> TokenStream<'b> {
    fn from_string<'a: 'b>(s: &'a String) -> TokenStream<'b> {
        TokenStream {
            offset: 0,
            s: s.as_bytes(),
        }
    }

    fn parse_number(&mut self) -> i32 {
        let mut val = 0;
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];
            if cur_byte < b'0' || cur_byte > b'9' {
                break;
            }
            let cur_digit = (cur_byte - b'0') as i32;
            val = val * 10 + cur_digit;
            self.offset += 1;
        }
        val
    }

    fn parse_identifier(&mut self) -> &'b str {
        let start_offset = self.offset;
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];

            // We don't check for "starting with digit"
            // case as it is already checked by next()
            // before entering this method.
            if cur_byte == b'(' || cur_byte == b')' || cur_byte == b' ' {
                break;
            }
            self.offset += 1;
        }
        assert!(start_offset < self.offset);
        std::str::from_utf8(&self.s[start_offset..self.offset])
            .expect("identifier to be correctly parsed")
    }
}

impl<'b> Iterator for TokenStream<'b> {
    type Item = Token<'b>;

    fn next(&mut self) -> Option<Self::Item> {
        while self.offset < self.s.len() {
            let cur_byte = self.s[self.offset];
            match cur_byte {
                b'(' => {
                    self.offset += 1;
                    return Some(Token::OpenParentheses);
                }
                b')' => {
                    self.offset += 1;
                    return Some(Token::CloseParentheses);
                }
                b'-' => {
                    self.offset += 1;
                    let num = self.parse_number();
                    return Some(Token::Number(-num));
                }
                b'0'..=b'9' => {
                    let num = self.parse_number();
                    return Some(Token::Number(num));
                }
                b'a'..=b'z' => {
                    let id = self.parse_identifier();
                    return Some(Token::Identifier(id));
                }
                b' ' => {
                    self.offset += 1;
                    continue;
                }
                ch => panic!("unexpected character: {}", ch as char),
            }
        }
        None
    }
}

struct Environment<'a> {
    sym_vals: Vec<HashMap<&'a str, i32>>,
}

impl<'a> Environment<'a> {
    fn new() -> Self {
        Environment { sym_vals: vec![] }
    }

    fn add_binding(&mut self, sym: &'a str, val: i32) {
        assert!(!self.sym_vals.is_empty());
        self.sym_vals.last_mut().map(|scope| scope.insert(sym, val));
    }

    fn lookup_symbol(&self, sym: &'a str) -> Option<i32> {
        self.sym_vals
            .iter()
            .rev()
            .find(|&tbl| tbl.contains_key(sym))
            .map(|tbl| *tbl.get(sym).unwrap())
    }

    fn create_scope(&mut self) {
        self.sym_vals.push(HashMap::new());
    }

    fn destroy_scope(&mut self) {
        self.sym_vals.pop();
    }
}

struct LispEvaluator<'b, 'a: 'b> {
    s: std::iter::Peekable<TokenStream<'a>>,
    env: Environment<'b>,
}

#[derive(Debug)]
struct EvalError(String);

const ADD_FUNC: &'static str = "add";
const MULTIPLY_FUNC: &'static str = "mult";
const LET_KEYWORD: &'static str = "let";

impl<'b, 'a> LispEvaluator<'b, 'a>
where
    'a: 'b,
{
    fn new(tok_stream: TokenStream<'a>) -> Self {
        let mut scope = Environment::new();
        scope.create_scope();
        LispEvaluator {
            s: tok_stream.peekable(),
            env: scope,
        }
    }

    // evaluate evaluates the Lisp expression written
    // according to the following grammar
    //
    // <expr> -> (add <expr> <expr>)
    //         | (mult <expr> <expr>)
    //         | (let [<ident> <expr>]+ <expr>)
    //         | <number>
    //         | <ident>
    fn evaluate(&mut self) -> Result<i32, EvalError> {
        self.evaluate_expr()
    }

    fn evaluate_expr(&mut self) -> Result<i32, EvalError> {
        let first_tok = self.s.next();
        match first_tok {
            Some(Token::OpenParentheses) => {
                let second_tok = self.s.next();
                let val = match second_tok {
                    Some(Token::Identifier(ADD_FUNC)) => self.evaluate_addition(),
                    Some(Token::Identifier(MULTIPLY_FUNC)) => self.evaluate_multiplication(),
                    Some(Token::Identifier(LET_KEYWORD)) => self.evaluate_let_block(),
                    Some(tok) => Err(EvalError(format!(
                        "unknown token: {:?}, expected add,mult,let",
                        tok
                    ))),
                    None => Err(EvalError(format!("expected add,mult,let, but got nothing"))),
                };
                self.ensure_closing_parentheses().and(val)
            }
            Some(Token::Identifier(ident)) => self.evaluate_identifier(ident),
            Some(Token::Number(n)) => Ok(n),
            Some(tok) => Err(EvalError(format!("unexpected token: {:?}", tok))),
            None => Err(EvalError(format!(
                "no token found for evaluating expression"
            ))),
        }
    }

    fn evaluate_addition(&mut self) -> Result<i32, EvalError> {
        let left_val = self.evaluate()?;
        let right_val = self.evaluate()?;
        Ok(left_val + right_val)
    }

    fn evaluate_multiplication(&mut self) -> Result<i32, EvalError> {
        let left_val = self.evaluate()?;
        let right_val = self.evaluate()?;
        Ok(left_val * right_val)
    }

    fn evaluate_let_block(&mut self) -> Result<i32, EvalError> {
        // The first one would be an identifier and the second
        // one would be an expression. The last one would be an
        // expression followed by closing parentheses. An identifier
        // is also a valid expression and hence that works.
        self.env.create_scope();
        let expr_result;
        loop {
            let ident_tok = self.s.peek();
            match ident_tok {
                Some(&Token::Identifier(ident)) => {
                    self.s.next();
                    let next_tok = self.s.peek();

                    // If it is an identifier followed by closing parentheses
                    // then this should be treated as an expression and should
                    // be evaluated as such. If it not a closing parentheses,
                    // then it should be evaluated as an expression and should
                    // be added to the environment.
                    if next_tok == Some(&Token::CloseParentheses) {
                        expr_result = self.evaluate_identifier(ident);
                        break;
                    } else {
                        let expr_val = self.evaluate_expr()?;
                        self.env.add_binding(ident, expr_val);
                    }
                }
                Some(_) => {
                    // If there is something else in a position where identifier
                    // is expected, then this must be an expression evaluated in
                    // the context of all the let bindings seen so far.
                    expr_result = self.evaluate_expr();
                    break;
                }
                None => {
                    expr_result = Err(EvalError(format!(
                        "expected expression at the end of let block"
                    )));
                    break;
                }
            };
        }
        self.env.destroy_scope();
        expr_result
    }

    fn evaluate_identifier(&mut self, ident: &'a str) -> Result<i32, EvalError> {
        self.env
            .lookup_symbol(ident)
            .ok_or(EvalError(format!("cannot lookup symbol: {}", ident)))
    }

    fn ensure_closing_parentheses(&mut self) -> Result<(), EvalError> {
        let tok = self.s.next();
        if tok == Some(Token::CloseParentheses) {
            Ok(())
        } else {
            Err(EvalError(format!(
                "expected closing parentheses, got {:?}",
                tok
            )))
        }
    }
}

fn evaluate(s: String) -> i32 {
    let stream = TokenStream::from_string(&s);
    let mut evaluator = LispEvaluator::new(stream);
    evaluator.evaluate().expect("expected i32, but got error")
}
{{< /highlight >}}