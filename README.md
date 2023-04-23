Download Link: https://assignmentchef.com/product/solved-assignment-2-command-interpreter-cs429
<br>
In this assignment, you will be writing an <em>interpreter </em>for EEL <a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a>, a “little language” of expressions. A calculator program is an interpreter. A Linux shell <a href="#_ftn2" name="_ftnref2"><sup>[2]</sup></a> is a <em>command interpreter </em>that you have used to execute Linux commands. You may also have used interpreters for programming languages such as Python or Matlab.

The focus of this lab is to give you practice in the style of C programming you will need to be able to do proficiently, especially for the later assignments in the class. Some of the skills developed are:

<ul>

 <li>Explicit memory management using the malloc and free calls, which is very different from what you have used in Java.</li>

 <li>Creating and manipulating pointer-based data structures: trees, linked lists, and hash tables. While pointers in C are similar to references in Java, there are key differences between them that you need to internalize.</li>

 <li>Working with strings. There is no String class in C, as you will find.</li>

 <li>Implementing robust code that operates correctly with invalid arguments, such as NULL pointers.</li>

</ul>

<h1>1          Download and setup</h1>

On your local machine, download the file cilab.tar from the Assignment 2: CI folder in the Files tab of the Canvas page for the class. Then use the scp command to copy the file over to your account on a lab machine.

For this assignment, you will also need to create a new repository in your GitHub account. Follow the instructions <a href="https://github.com/new">here.</a> Create a new repository, name it CI-Lab, and mark it private. You should know how to do this from Assignment 1.

<h1>2          Details of the assignment</h1>

At their core, all interpreters go through a common three-phase cycle of activities, called the Read-Eval-Print Loop (REPL for short). They repeatedly <em>read </em>some input (typically interactive, but possibly from a file or from input redirection), <em>evaluate </em>that input (do something with it), and <em>print </em>the result of that evaluation (either an actual value or a message produced as a side-effect of evaluation). In the Linux shell example, the input is the command that you type in at the prompt (let’s say it’s ls). The shell program parses this input, validates it as a known command, and proceeds to execute it. <a href="#_ftn3" name="_ftnref3"><sup>[3]</sup></a> The results of executing the command (in this case, the desired listing of files in the current directory) is printed to the terminal window, and the shell resumes waiting at its prompt for the next command.

Your task over the course of this assignment will be to write a similar (but much simpler) interpreter for EEL in stages. We will provide most of the code for reading and printing, so your focus will be on the evaluation part of REPL.

<ul>

 <li>Your first interpreter will handle a core language called EEL-0 with only two data types (integers and</li>

</ul>

Booleans), literals (i.e., constants, not variables) and a handful of unary, binary, and ternary operations.

<ul>

 <li>Next, you will expand the language to EEL-1 by adding a third type (strings) to the language, along with a handful of (overloaded) unary and binary operations.</li>

 <li>Finally, you will add named variables to create the language EEL-2.</li>

</ul>

4.1     Background

Inside the EEL interpreter, you will represent the expression that the user types in at the terminal as a tree.

For example, if you typed in the character string <em>I</em><sub>1 </sub>= ((32+100)*(16-4)) to the prompt, the internal representation would look like this: <em>T</em><sub>1 </sub>=               *             .

32     100     16     4

You probably realize by now that what we are doing is basically the PEMDAS (or BODMAS) rule that you learned in grade school. However, we have made things easier for you by engineering the EEL grammar to require that all expressions be fully parenthesized, which removes any issues of operator precedence. Thus, the input string would not be parsed as being a valid EEL expression, while the string would have the internal representation – .

100     16

We transform the input character string representation (the “concrete syntax”) into the internal tree representation (the “abstract syntax tree” or AST) in two steps. First, we take the input line and break it into atomic chunks of the language (“tokens”). Then, we process this stream of tokens and convert it into the tree. For now, think of a stream as an abstract data type similar to a sequence, but with limited ways of accessing elements. You can access a stream element only once, and in sequential order. In the example above, the intermediate token stream generated from input string <em>I</em><sub>1 </sub>would be

<em>S</em><sub>1 </sub>= [ ( ( 32 + 100 ) * ( 16 – 4 ) ) ],

with each token enclosed in a box and the entire stream delimited by square brackets.

<h2>4.1.1      Converting a line of text into tokens</h2>

The process of going from the input character string to the sequence of tokens is called <em>lexical analysis </em>(aka “lexing” or “tokenization”). This process involves some details that would be a distraction right now, so we are providing the code for this module in the file lex.c. The module exports two variables this_token and next_token and two functions init_lexer() and advance_lexer(). The specific token types that are returned by the lexer module are listed in the file tokens.h.

Why do we provide a peek (aka “lookahead”) at the next token in the stream? Consider the input line

<em>I</em><sub>2 </sub>= ((32+(10*10))*(16-4)),

which turns into the token stream

<em>S</em><sub>2 </sub>= [ ( ( 32 + ( 10 * 10 ) ) * ( 16 – 4 ) ) ]

and finally into the AST

<em>T</em><sub>2 </sub>=                        *                  ,

10     10

whose <em>structure </em>is recognizably different from that of <em>T</em><sub>1</sub>. <a href="#_ftn4" name="_ftnref4"><sup>[4]</sup></a> The difference between the AST structures is in the right child of the + node: a single leaf node for <em>T</em><sub>1</sub>, a subtree for <em>T</em><sub>2</sub>. The two token streams differ starting from the fifth token: 100 for <em>S</em><sub>1</sub>, ( for <em>S</em><sub>2</sub>. We need to peek at this token before we consume it in order to choose the correct action to build the AST. This is another piece of grammar engineering: a <em>single </em>lookahead token will suffice to build the correct AST for an EEL expression.

One final point: whitespace is largely insignificant in EEL. <a href="#_ftn5" name="_ftnref5"><sup>[5]</sup></a> So one function of the lexer is to strip out ornamental whitespace (spaces or tabs) as it produces the token stream. This means that the input string

<em>I</em><sub>1 </sub>and the input string ( ( 32 + ( 10 * 10 ) ) * ( 16 – 4 ) ) (with additional whitespace characters, indicated as ) would both result in the same token stream <em>S</em><sub>2</sub>.

<h2>4.1.2       Building an AST from a stream of tokens</h2>

There are two key differences between the representations <em>S</em><sub>2 </sub>and <em>T</em><sub>2</sub>. First, <em>S</em><sub>2 </sub>is linear, while <em>T</em><sub>2 </sub>is nonlinear. Second, <em>T</em><sub>2 </sub>is missing a number of tokens—specifically, the tokens ( and ) . The purpose of these parentheses was to linearize the nonlinear tree structure while retaining the relationships between different parts of the expression. Once we build the tree representation, we can safely discard these tokens. This is why the tree is called an <em>abstract </em>syntax tree.

The structure of all possible ASTs that are syntactically valid EEL expressions is specified by a <em>grammar</em>, much like the grammar for a natural language. A grammar <a href="#_ftn6" name="_ftnref6"><sup>[6]</sup></a> is simply an inductive way of defining infinite sets that have a natural tree structure.

The way you are going to build the AST is using a technique called <em>recursive descent </em>parsing, which is a way of consuming the token stream token-by-token and building the tree in a synchronized manner, driven by the rules of the grammar. Here is a grammar rule (aka a “production”) for EEL:

<em>Expr </em>→ ( <em>Expr </em>+ <em>Expr </em>) <em>.</em>

This rule says that you can create the AST for a larger expression (the red one on the left of the →) by combining the subtrees of two smaller expressions (the blue and purple ones on the right of the →) into a tree with the + token, thus: + . To terminate the recursion, we have rules like this:

<em>Expr          </em><em>Expr</em>

<em>Expr </em>→ <em>Literal</em>, which says that if your current token is a literal, it is also an expression.

How do you know which rule to use? This is where the grammar engineering comes in. If you look at the two rules above, you will realize that the very first token (either this_token or next_token from the lexer module) tells you unambiguously what action to take. If it is the ( token, you use the first rule; if next_token is a literal, you use the second rule; if it is anything else, that is an error condition.

Table 1 shows how the input token sequence <em>S</em><sub>1 </sub>is traversed step-by-step and the AST <em>T</em><sub>1 </sub>is assembled. The complete EEL grammar is in Appendix A.

<h2>4.1.3       Inferring the types of AST nodes</h2>

Unlike Java or C, we don’t explicitly specify the types of the variables or constants in EEL. And if you look at the definition of <em>Literal </em>in the EEL grammar, you will see that it encompasses integer constants, Boolean constants, and string constants (in EEL-1 and EEL-2). This means that it is possible to create an AST for Table 1: Step-by-step snapshots of AST construction for the user input <em>I</em><sub>1</sub>. The current token is in red, the lookahead token in blue, and processed tokens in green. The AST shown above a step is in its state before consuming the current token, while the AST shown below is in its state after consuming the current token.

<table width="342">

 <tbody>

  <tr>

   <td width="31">Step</td>

   <td colspan="2" width="27"></td>

   <td colspan="3" width="52"></td>

   <td colspan="7" width="125">Token stream</td>

   <td width="16"></td>

   <td colspan="2" width="25"></td>

   <td width="65">Current token</td>

  </tr>

  <tr>

   <td width="31"></td>

   <td colspan="2" width="27"></td>

   <td colspan="3" width="52"></td>

   <td colspan="7" width="125"><em>Expr</em></td>

   <td width="16"></td>

   <td colspan="2" width="25"></td>

   <td width="65"></td>

  </tr>

  <tr>

   <td width="31">1</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">LPAREN</td>

  </tr>

  <tr>

   <td width="31"></td>

   <td width="13"></td>

   <td width="15"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="25"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="15"></td>

   <td width="11"></td>

   <td width="65"></td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">2</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">LPAREN</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">3</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">Number</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">4</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">PLUS</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">5</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">Number</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">6</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">RPAREN</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">7</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">TIMES</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">8</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">LPAREN</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">9</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">Number</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">10</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">MINUS</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td width="31">11</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">Number</td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td colspan="5" width="95"></td>

   <td width="16"></td>

   <td colspan="3" width="57">       32        100</td>

   <td colspan="4" width="68">     16        4</td>

   <td width="16"></td>

   <td colspan="2" width="25"></td>

   <td width="65"></td>

  </tr>

  <tr>

   <td width="31">12</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">RPAREN</td>

  </tr>

  <tr>

   <td width="31"></td>

   <td width="13"></td>

   <td width="15"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="25"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="15"></td>

   <td width="11"></td>

   <td width="65"></td>

  </tr>

 </tbody>

</table>

<table width="342">

 <tbody>

  <tr>

   <td colspan="5" width="95"></td>

   <td width="16"></td>

   <td colspan="3" width="57">       32        100</td>

   <td colspan="4" width="68">     16        4</td>

   <td width="16"></td>

   <td colspan="2" width="25"></td>

   <td width="65"></td>

  </tr>

  <tr>

   <td width="31">13</td>

   <td width="13">[</td>

   <td width="15">(</td>

   <td width="16">(</td>

   <td width="21">32</td>

   <td width="16">+</td>

   <td width="25">100</td>

   <td width="16">)</td>

   <td width="16">*</td>

   <td width="16">(</td>

   <td width="21">16</td>

   <td width="16">–</td>

   <td width="16">4</td>

   <td width="16">)</td>

   <td width="15">)</td>

   <td width="11">]</td>

   <td width="65">RPAREN</td>

  </tr>

  <tr>

   <td width="31"></td>

   <td width="13"></td>

   <td width="15"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="25"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="21"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="16"></td>

   <td width="15"></td>

   <td width="11"></td>

   <td width="65"></td>

  </tr>

 </tbody>

</table>

–

–

something like 2+true, which is meaningless because you can’t add an integer to a Boolean. This is like an English sentence “The ball threw the boy”, which is grammatical (syntactically correct) but nonsensical (semantically incorrect). So, after creating the AST but before evaluating it, you need to validate that it is well-formed. This process is called <em>type inference</em>, and you will do this by a postorder traversal of the AST.

You will know the type of a leaf node based on its value (an integer constant, the Boolean values true and false, or a string constant delimited by a pair of double-quote characters). For an internal node, you first (recursively) infer the types of its children; then you figure out whether the operation represented by that node is compatible with the types of its arguments; if it is, you infer the type of that node. For example, the + operation is defined only on pairs of integers and pairs of strings <a href="#_ftn7" name="_ftnref7"><sup>[7]</sup></a> (EEL-1 onwards), so both of its children must be of one of those types, and the output is also the corresponding type. Any other combination of input types signifies an error. Keep in mind operations like ˜ , where the output type can be something different from the input types; and the ternary operation ?: , for which not all inputs have the same type.

While EEL is an implicitly-typed language, we also want it to be statically typed. By this we mean that an expression like ((20 &lt; 15) ? 3 : true) will fail during type inference.

<h2>4.1.4       Computing the value of AST nodes</h2>

At this point of the process, you have converted the input characters into an AST and validated the wellformedness of the AST with respect to type. You are now ready to (recursively) associate values with each subtree of the AST. This is again done with a postorder traversal of the AST, except that you will be computing values at each internal node rather than inferring types.

<h2>4.1.5      Error handling</h2>

Not all user inputs are valid, and you will need to handle errors. Given the multi-step nature of the expression evaluation process, errors can happen in multiple places. The principle of error handling you will use is this: <em>Handle the error as early as you can in the process, but no earlier. </em>Typically, these failures means that none of the following stages can be executed meaningfully, so you will print an error message and return an appropriate indicator to your caller routine.

We classify errors by the earliest time that we can recognize them. Here are the classes.

<ol>

 <li><em>Lexical errors </em>are those that cause lexical analysis to fail to create a valid token, e.g., an unrecognized character or an invalid syntax for an identifier. The supplied lexer module handles such errors for you.</li>

 <li><em>Syntactic errors </em>are those that cause the parser fail to create a valid AST. For instance, the token stream 1 + 2 is not a valid production (it needs to be fully parenthesized); neither is ) 1 +</li>

</ol>

2 ( (the parentheses occur in the wrong order) or ( 1 + 2 (the parentheses aren’t matched). Your parser module must handle these errors.

<ol start="3">

 <li><em>Type inference errors </em>are those that fail to infer valid types for the AST. <a href="#_ftn8" name="_ftnref8"><sup>[8]</sup></a> Examples include *</li>

</ol>

32     true

and                  ?:                    .

true

42     50

<ol start="4">

 <li><em>Evaluation errors </em>are those that fail to produce a meaningful value for a subtree. The only cases where this happens are for the integer operations of division and remainder, where a divisor value of zero causes problems; and for the string replication operation *, where the second argument (an integer) cannot have a negative value.</li>

</ol>

4.2     What you will code

Before you do anything else, update line 15 of the file interface.c with your name and UT EID as specified. Save the file; you are done with changes to this file.

Next, study the five files err_handler.h, node.h, token.h, type.h, and value.h carefully. These provide the documentation of the data structures you will be using. It may also be helpful to study the files err_handler.c and lex.c to understand the interface for handling errors.

Now you are ready to start writing your own code. Search for the string (STUDENT TODO) in the files parse.c and eval.c. These are the routines that you need to write, and the comments above the routines tell you what you have to do. Start with the EEL-0 language before pushing on to EEL-1 and eventually EEL-2. In fact, you should start with the examples given in tests/test_format.txt and tests/test_simple.txt. When you are ready to move on to EEL-2, you will also need to fill in some routines in variable.c.

One more helper routine that is useful for debugging is print_tree(). It allows you to see a simple visualization of the AST. TAs will cover the details of this routine in the Friday sections.

In order to build an executable that you can run, type the command make ci at the Linux prompt. This will run the C compiler gcc on the necessary files and put everything together into a binary file called ci. We have also provided you a pre-built reference implementation of the interpreter, which is in the binary file ci_reference. To run either of these binaries, simply type its name at the Linux prompt, and you will be running in interactive mode. If you want the interpreter to read input from a text file called <em>foo</em>, you can do that by specifying the command-line parameter -i <em>foo</em>. If you want the interpreter to write output to a text file called <em>bar</em>, you can do that by specifying the command-line parameter -o <em>bar</em>. You can use the driver.sh shell script <a href="#_ftn9" name="_ftnref9"><sup>[9]</sup></a> to check the output of your implementation against the reference one.

The tests subdirectory contains test files. If you want to write your own test files, look at the examples in here. The format is quite simple: one expression per line, with the last line being the string @q.

<h1>3          Handing in your assignment</h1>

You will hand in this assignment using Gradescope. For code, link your GitHub account to Gradescope, and select the CI-Lab repository when submitting the assignment. Make sure all files are up to date (i.e., committed) on GitHub before you hand in the assignment. For the writeup, upload the PDF file to Gradescope.

<h1>4          Evaluation</h1>

This assignment is worth 9% of the total course grade.

Each of the checkpoints will contribute 1%. You will be considered to have passed checkpoint 1 if your code compiles without error and passes the tests in tests/test_simple.txt. You will be considered to have passed checkpoint 2 if your code compiles without error and passes all the integer and string tests in subdirectory test, and you have turned in one nontrivial test case for EEL-0 in tests/test_custom.txt.

The final hand-in will contribute 7%. Your code must compile without error. The correctness of your solution will be determined by test cases covering the EEL-0, EEL-1, and EEL-2 language levels. This will include some tests that will be provided to you as well as other blind tests that you can run on Gradescope.

<h1>A         The EEL Grammar</h1>

<table width="651">

 <tbody>

  <tr>

   <td width="35">Id</td>

   <td colspan="22" width="251">Production</td>

   <td width="78">EEL level(s)</td>

   <td colspan="7" width="146">AST Fragment</td>

   <td colspan="3" width="140">Comments</td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P1</td>

   <td colspan="11" rowspan="2" width="148"><em>Root </em>→ <em>Expr</em></td>

   <td colspan="2" width="21">NL</td>

   <td colspan="4" rowspan="2" width="21"></td>

   <td colspan="5" rowspan="2" width="60"></td>

   <td rowspan="2" width="78">0, 1, 2</td>

   <td colspan="7" rowspan="2" width="146"><em>Root</em><em>Expr</em></td>

   <td rowspan="2" width="8"></td>

   <td width="23">NL</td>

   <td rowspan="2" width="109">is newline (’
’).</td>

  </tr>

  <tr>

   <td colspan="2" width="21"></td>

   <td width="23"></td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P2</td>

   <td colspan="7" rowspan="2" width="120"><em>Root </em>→ <em>Expr</em></td>

   <td colspan="3" width="16">#</td>

   <td colspan="5" rowspan="2" width="41"><em>Format</em></td>

   <td colspan="2" width="14">N</td>

   <td width="9">L</td>

   <td colspan="4" rowspan="2" width="52"></td>

   <td rowspan="2" width="78">0, 1, 2</td>

   <td colspan="7" rowspan="2" width="146"><em>Root</em><em>Expr      Format</em></td>

   <td rowspan="2" width="8"></td>

   <td width="23">NL</td>

   <td rowspan="2" width="109">is newline (’
’).</td>

  </tr>

  <tr>

   <td colspan="3" width="16"></td>

   <td colspan="2" width="14"></td>

   <td width="9"></td>

   <td width="23"></td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P3</td>

   <td colspan="13" rowspan="2" width="170"><em>Root </em>→ <em>Var </em>= <em>Expr</em></td>

   <td colspan="4" width="21">NL</td>

   <td colspan="5" rowspan="2" width="60"></td>

   <td rowspan="2" width="78">2</td>

   <td colspan="7" rowspan="2" width="146"><em>Root</em><em>Var      Expr</em></td>

   <td colspan="3" rowspan="2" width="140">An initialized variable.</td>

  </tr>

  <tr>

   <td colspan="4" width="21"></td>

  </tr>

  <tr>

   <td width="35">P4</td>

   <td colspan="21" width="223"><em>Expr </em>→ <em>Literal</em></td>

   <td width="28"></td>

   <td width="78">0, 1, 2</td>

   <td colspan="7" width="146"><em>Literal</em></td>

   <td colspan="3" width="140"></td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P5</td>

   <td rowspan="2" width="71"><em>Expr </em>→</td>

   <td width="16">(</td>

   <td colspan="4" rowspan="2" width="30"><em>Expr</em></td>

   <td colspan="3" width="15">?</td>

   <td colspan="11" rowspan="2" width="75"><em>Expr </em>: <em>Expr</em></td>

   <td width="16">)</td>

   <td rowspan="2" width="28"></td>

   <td rowspan="2" width="78">0, 1, 2</td>

   <td colspan="7" rowspan="2" width="146">?:<em>Expr      Expr      Expr</em></td>

   <td colspan="3" rowspan="2" width="140">Ternary choice operator.</td>

  </tr>

  <tr>

   <td width="16"></td>

   <td colspan="3" width="15"></td>

   <td width="16"></td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P6</td>

   <td colspan="2" rowspan="2" width="87"><em>Expr </em>→</td>

   <td colspan="2" width="15">(</td>

   <td colspan="13" rowspan="2" width="89"><em>Expr Binop Expr</em></td>

   <td colspan="3" width="16">)</td>

   <td rowspan="2" width="16"></td>

   <td rowspan="2" width="28"></td>

   <td rowspan="2" width="78">0, 1, 2</td>

   <td colspan="7" rowspan="2" width="146"><em>Binop</em><em>Expr      Expr</em></td>

   <td colspan="3" rowspan="2" width="140">Binary operations.</td>

  </tr>

  <tr>

   <td colspan="2" width="15"></td>

   <td colspan="3" width="16"></td>

  </tr>

  <tr>

   <td rowspan="2" width="35">P7</td>

   <td colspan="4" rowspan="2" width="102"><em>Expr </em>→</td>

   <td colspan="2" width="15">(</td>

   <td colspan="9" rowspan="2" width="60"><em>Unop Expr</em></td>

   <td colspan="2" width="14">)</td>

   <td colspan="4" rowspan="2" width="33"></td>

   <td rowspan="2" width="28"></td>

   <td rowspan="2" width="78">0, 1, 2</td>

   <td colspan="7" rowspan="2" width="146"><em>Unop</em><em>Expr</em></td>

   <td colspan="3" rowspan="2" width="140">Unary operations.</td>

  </tr>

  <tr>

   <td colspan="2" width="15"></td>

   <td colspan="2" width="14"></td>

  </tr>

  <tr>

   <td width="35">P8</td>

   <td colspan="6" width="117"><em>Expr </em>→</td>

   <td colspan="3" width="15">(</td>

   <td colspan="3" width="30"><em>Expr</em></td>

   <td colspan="3" width="15">)</td>

   <td colspan="6" width="46"></td>

   <td width="28"></td>

   <td width="78">0, 1, 2</td>

   <td colspan="7" width="146"><em>Expr</em></td>

   <td colspan="3" width="140">Extra parentheses.</td>

  </tr>

  <tr>

   <td width="35">P9</td>

   <td colspan="21" width="223"><em>Expr </em>→ <em>Var</em></td>

   <td width="28"></td>

   <td width="78">2</td>

   <td colspan="7" width="146"><em>Var</em></td>

   <td colspan="3" width="140">A named variable.</td>

  </tr>

  <tr>

   <td width="35">P10</td>

   <td colspan="22" width="251"><em>Binop </em>→[+-*/%&amp;|&lt;&gt;˜]</td>

   <td width="78">0, 1, 2</td>

   <td width="8"></td>

   <td width="15"><em>x</em></td>

   <td colspan="5" width="122">, <em>x</em>∈[+-*/%&amp;|&lt;&gt;˜]</td>

   <td colspan="3" width="140">Valid binary operations.</td>

  </tr>

  <tr>

   <td width="35">P11</td>

   <td colspan="22" width="251"><em>Unop </em>→[_!]</td>

   <td width="78">0, 1, 2</td>

   <td colspan="3" width="37"></td>

   <td width="15"><em>x</em></td>

   <td colspan="3" width="94">, <em>x</em>∈[_!]</td>

   <td colspan="3" width="140">Valid unary operations.</td>

  </tr>

  <tr>

   <td width="35">P12</td>

   <td colspan="14" width="172"><em>Literal </em>→ <em>Num</em></td>

   <td colspan="8" width="80"></td>

   <td width="78">0, 1, 2</td>

   <td colspan="7" width="146"><em>Num</em></td>

   <td colspan="3" width="140">Integral constant values.</td>

  </tr>

  <tr>

   <td width="35">P13</td>

   <td colspan="8" width="131"><em>Literal </em>→</td>

   <td colspan="6" width="41">true</td>

   <td colspan="8" width="80"></td>

   <td width="78">0, 1, 2</td>

   <td colspan="7" width="146">true</td>

   <td colspan="3" width="140">Boolean constant true.</td>

  </tr>

  <tr>

   <td width="35">P14</td>

   <td colspan="8" width="131"><em>Literal </em>→</td>

   <td colspan="6" width="41">false</td>

   <td colspan="8" width="80"></td>

   <td width="78">0, 1, 2</td>

   <td colspan="7" width="146">false</td>

   <td colspan="3" width="140">Boolean constant false.</td>

  </tr>

  <tr>

   <td width="35">P15</td>

   <td colspan="14" width="172"><em>Literal </em>→ <em>String</em></td>

   <td colspan="8" width="80"></td>

   <td width="78">1, 2</td>

   <td colspan="7" width="146"><em>String</em></td>

   <td colspan="3" width="140">A string contant.</td>

  </tr>

  <tr>

   <td width="35">P16</td>

   <td colspan="22" width="251"><em>Var </em>→ <em>Identifier</em></td>

   <td width="78">2</td>

   <td colspan="6" width="81"><em>Identi</em></td>

   <td width="65"><em>fier</em></td>

   <td colspan="3" width="140"></td>

  </tr>

  <tr>

   <td width="35">P17</td>

   <td colspan="22" width="251"><em>Num </em>→[0-9]+</td>

   <td width="78">0, 1, 2</td>

   <td colspan="5" width="65"></td>

   <td width="15"><em>x</em></td>

   <td width="65"></td>

   <td colspan="3" width="140"><em>x </em>is the numerical value.</td>

  </tr>

  <tr>

   <td width="35">P18</td>

   <td colspan="3" width="95"><em>String </em>→</td>

   <td colspan="2" width="16">“</td>

   <td colspan="11" width="78">[’ ’-’˜’]*</td>

   <td colspan="3" width="16">“</td>

   <td colspan="3" width="47"></td>

   <td width="78">1, 2</td>

   <td colspan="5" width="65"></td>

   <td width="15"><em>x</em></td>

   <td width="65"></td>

   <td colspan="3" width="140"><em>x </em>is the string value.</td>

  </tr>

  <tr>

   <td width="35">P19</td>

   <td colspan="22" width="251"><em>Identifier </em>→[a-zA-Z]([a-zA-Z][0-9])*</td>

   <td width="78">2</td>

   <td colspan="5" width="65"></td>

   <td width="15"><em>x</em></td>

   <td width="65"></td>

   <td colspan="3" width="140"><em>x </em>is the id name.</td>

  </tr>

  <tr>

   <td width="35">P20</td>

   <td colspan="22" width="251"><em>Format </em>→[dxXbB]</td>

   <td width="78">0, 1, 2</td>

   <td colspan="5" width="65"></td>

   <td width="15"><em>x</em></td>

   <td width="65"></td>

   <td colspan="3" width="140"><em>x </em>is the format specifier.</td>

  </tr>

  <tr>

   <td width="34"></td>

   <td width="71"></td>

   <td width="16"></td>

   <td width="8"></td>

   <td width="7"></td>

   <td width="8"></td>

   <td width="6"></td>

   <td width="3"></td>

   <td width="11"></td>

   <td width="1"></td>

   <td width="4"></td>

   <td width="13"></td>

   <td width="14"></td>

   <td width="8"></td>

   <td width="2"></td>

   <td width="5"></td>

   <td width="12"></td>

   <td width="2"></td>

   <td width="9"></td>

   <td width="5"></td>

   <td width="3"></td>

   <td width="16"></td>

   <td width="28"></td>

   <td width="78"></td>

   <td width="8"></td>

   <td width="15"></td>

   <td width="13"></td>

   <td width="15"></td>

   <td width="13"></td>

   <td width="15"></td>

   <td width="65"></td>

   <td width="8"></td>

   <td width="23"></td>

   <td width="109"></td>

  </tr>

 </tbody>

</table>

<ul>

 <li>In production P5, evaluation proceeds as in C or Java: the condition is first evaluated, and only one choice is evaluated depending on the value of the condition.</li>

 <li>In productions P6 and P10, the interpretation of the binary operation depends on the types of the input operands.

  <ul>

   <li>For integers, +-*/% are sum, difference, product, quotient, and remainder; and &lt;&gt;˜ are the comparisons less-than, greater-than, and equal-to.</li>

   <li>For Booleans, &amp; is AND and | is inclusive-OR.</li>

   <li>For strings, + is concatenation; * is repetition (with the second argument being an integer); and &lt;&gt;˜ compare the lexicographic ordering of two strings.</li>

   <li>No other combination is legal.</li>

  </ul></li>

 <li>In productions P7 and P11, the interpretation of the unary operation depends on the type of the input operand.

  <ul>

   <li>For integers, _ is negation.</li>

   <li>For Booleans, ! is NOT.</li>

   <li>For strings, _ is string reversal.</li>

   <li>No other combination is legal.</li>

  </ul></li>

 <li>Productions P17–P20 use standard grep syntax. is the whitespace character.</li>

 <li>In production P18, the characters making up the string can be any printable characters. In the sevenbut ASCII character set, these are 0x20 (’’) through 0x7E (’˜’). The function isprint() in &lt;ctype.h&gt; tests for this property.</li>

 <li>The default format for printing integers and Booleans is d, which prints them as decimals. The format specifiers x and X cause integers to be printed in hexadecimal. while b and B causes Booleans to be printed as named values. Strings are always printed as character strings.</li>

</ul>

<a href="#_ftnref1" name="_ftn1">[1]</a> EEL stands for “Expression Evaluation Language”.

<a href="#_ftnref2" name="_ftn2">[2]</a> There is nothing particular special about a shell. There are multiple shells available in Linux, and you can choose which to use. You can even write your own shell.

<a href="#_ftnref3" name="_ftn3">[3]</a> The actual method of execution is hairy. You will learn about it in C S 439.

<a href="#_ftnref4" name="_ftn4">[4]</a> Both <em>T</em><sub>1 </sub>and <em>T</em><sub>2 </sub>evaluate to the same value. That is an orthogonal matter.

<a href="#_ftnref5" name="_ftn5">[5]</a> Not always, however. Consider the difference between inputs 32 and 3 2. The first input produces the token stream [ 32 ], while the second produces the token stream [ 3 2 ].

<a href="#_ftnref6" name="_ftn6">[6]</a> Technically, this is called a context-free grammar. There are other types of grammars.

<a href="#_ftnref7" name="_ftn7">[7]</a> Mathematically speaking, the operator + is overloaded, with signatures + : <em>Int </em>× <em>Int </em>→ <em>Int </em>and + : <em>String </em>× <em>String </em>→ <em>String</em>.

<a href="#_ftnref8" name="_ftn8">[8]</a> Since there weren’t any syntactic errors, you have a valid AST.

<a href="#_ftnref9" name="_ftn9">[9]</a> A shell script is simply a program written in the “little language” that the shell interprets. It is a text file. Feel free to study it if you are curious.