# Automata Theory Monsoon 2026 — Programming Assignment

**Roll Number:** 2025101029  
**Language:** Python 3  

---

## Directory Structure

```text
2025101029/
├── Question-1/
│   └── regexpert.py        # Regex engine (Thompson's NFA & Subset Construction DFA)
├── Question-2/
│   └── q2.py               # Recursive-descent parser for robot-control language
└── README.md               # Explanation of solutions and usage
```

---

## Question 1: RegExpert (`regexpert.py`)

`regexpert.py` is an end-to-end regular expression engine implemented from first principles without relying on built-in regex libraries (`re`) or backtracking algorithms.

### 1. Architecture & Pipeline

The regex engine processes expressions and input text through a 5-stage pipeline:

1. **Validation & Tokenization:**
   - Scans the raw regular expression string and tokenizes it into operands, operators, and character classes.
   - Automatically inserts implicit concatenation operators (`.`) between adjacent tokens (such as between literals, grouped expressions, character classes, and quantifiers).
   - Handles escaped literals (`\.`, `\*`, `\+`, `\(`, `\)`, `\\`, etc.) as literal character operands.
   - Parses character classes (`[abc]`) and character ranges (`[a-z]`, `[0-9]`), expanding valid ranges into character sets.
   - Detects and rejects malformed inputs with descriptive parse errors to standard error (`stderr`), exiting with non-zero status:
     - Missing operands (e.g., `a|`, `|a`, `a||b`)
     - Quantifiers without operands or stacked quantifiers (e.g., `*a`, `a**`, `a+*`)
     - Unmatched or empty parentheses (e.g., `(`, `)`, `()`)
     - Unterminated or invalid character classes/ranges (e.g., `[`, `[z-a]`)
     - Unsupported operators (e.g., unescaped `.`, `[^a]`)
     - Whitespace and non-ASCII characters

2. **Infix to Postfix Conversion (Shunting-Yard Algorithm):**
   - Converts infix regular expressions into postfix notation using Dijkstra's Shunting-Yard algorithm.
   - Enforces correct operator precedence and associativity:
     - Precedence 3 (Highest): Quantifiers (`*`, `+`, `?`)
     - Precedence 2: Concatenation (`.`)
     - Precedence 1 (Lowest): Alternation / Union (`|`)

3. **Thompson's NFA Construction:**
   - Transforms the postfix token stream into a Non-deterministic Finite Automaton (NFA) with $\varepsilon$-transitions.
   - Each NFA fragment maintains a single entry state (`start`) and a single exit state (`accept`).
   - Implemented constructs:
     - **Literal / Character Class:** Single transition from `start` to `accept` for each matched character.
     - **Concatenation ($AB$):** Connects the accept state of fragment $A$ to the start state of fragment $B$ via an $\varepsilon$-transition.
     - **Union ($A \mid B$):** Creates new start and accept states; branches $\varepsilon$-transitions to $A$ and $B$, and merges their accept states into the new accept state.
     - **Kleene Star ($A^*$):** Adds $\varepsilon$-transitions allowing zero occurrences (bypass) or repeated occurrences (loopback).
     - **Plus ($A^+$):** Requires at least one match through fragment $A$ before looping back.
     - **Question Mark ($A^?$):** Allows zero or one occurrence via a bypass $\varepsilon$-transition.

4. **Subset Construction (NFA to DFA Conversion):**
   - Transforms the NFA into an equivalent Deterministic Finite Automaton (DFA) using subset construction:
     - Computes $\varepsilon$-closure for state sets using iterative depth-first traversal.
     - The DFA start state is the $\varepsilon$-closure of the NFA start state.
     - Lazily explores reachable state subsets for each symbol in the active alphabet $\Sigma$.
     - Rejection/dead states are excluded from the state set.
     - A DFA state is marked accepting if its constituent NFA state set contains the NFA's accept state.

5. **Simulation & Matching:**
   - Splits standard input text by whitespace into words, keeping punctuation attached.
   - Simulates each word character by character on the DFA from the start state.
   - A word matches if and only if the entire word is consumed and the DFA halts in an accept state (strict whole-word match).
   - Matching words are printed to `stdout` in document order.

### 2. Debug Mode

When run with `--debug`, the DFA transition matrix is printed to `stderr` in CSV format before matching begins:
- Header row lists active alphabet characters sorted in ascending ASCII order.
- States are labeled with `->` (start state) and `*` (accept states). Dead transitions are marked with `-`.

---

## Question 2: Syntactic Analysis (`q2.py`)

`q2.py` implements a compiler front-end for a robot-control domain-specific language consisting of a lexer (FSA-based tokenizer) and a recursive-descent parser.

### 1. Language Grammar (CFG)

The syntax is governed by the following context-free grammar:

$$
\begin{aligned}
S &\to P \\
P &\to \text{statement } P \mid \varepsilon \\
\text{statement} &\to \text{Move} \mid \text{Turn} \mid \text{Set} \mid \text{Print} \mid \text{If} \\
\text{Move} &\to \mathbf{move} \ x \ \mathbf{;} \\
\text{Turn} &\to \mathbf{turn} \ \mathbf{left} \ \mathbf{;} \mid \mathbf{turn} \ \mathbf{right} \ \mathbf{;} \\
\text{Set} &\to \mathbf{set} \ \mathbf{id} \ \mathbf{=} \ x \ \mathbf{;} \\
\text{Print} &\to \mathbf{print} \ x \ \mathbf{;} \\
\text{If} &\to \mathbf{if} \ x \ \text{RelOp} \ x \ \text{Move} \ \text{ElsePart} \\
\text{ElsePart} &\to \mathbf{else} \ \text{Move} \mid \varepsilon \\
\text{RelOp} &\to \mathbf{<} \mid \mathbf{>} \mid \mathbf{==} \\
x &\to \mathbf{integer} \mid \mathbf{float} \mid \mathbf{id}
\end{aligned}
$$

### 2. Architecture & Implementation

1. **Lexical Analysis (Tokenization):**
   - Uses Finite State Automata (FSAs) to categorize input characters into tokens:
     - `IDENTIFIER`: Starts with a letter or underscore, followed by alphanumeric characters or underscores.
     - `KEYWORD`: Reserved keywords (`move`, `turn`, `left`, `right`, `set`, `print`, `if`, `else`).
     - `INTEGER`: Consecutive numeric digits.
     - `FLOAT`: Numbers containing a decimal point with digits on both sides.
     - `SYMBOL`: Valid punctuation symbols (`;`, `=`, `<`, `>`, `==`).
   - Raises `ValueError` on lexical errors (e.g. identifiers starting with numbers, malformed floats) without printing partial tokens.

2. **Syntactic Analysis (Recursive-Descent Parser):**
   - The parser function `checkGrammar(tokens)` validates token sequences against the CFG rules using LL(1) predictive parsing.
   - Dedicated parsing functions for each non-terminal:
     - `parse_x`: Matches `INTEGER`, `FLOAT`, or `IDENTIFIER`.
     - `parse_semicolon`: Matches `;`.
     - `parse_move`: Validates `move x ;`.
     - `parse_turn`: Validates `turn left ;` and `turn right ;`.
     - `parse_set`: Validates `set id = x ;`.
     - `parse_print`: Validates `print x ;`.
     - `parse_relop`: Validates relational operators `<`, `>`, `==`.
     - `parse_elsepart`: Validates `else Move` or $\varepsilon$.
     - `parse_if`: Validates `if x RelOp x Move ElsePart`.
     - `parse_P`: Iteratively parses a sequence of zero or more statements until end-of-tokens.

3. **Error Handling:**
   - **Lexical Errors:** Tokenization fails immediately; `ValueError` is raised with a descriptive message and no tokens are printed.
   - **Syntactic Errors:** When tokenization succeeds but the sequence violates the grammar, all recognized tokens are printed to `stdout` first, followed by a descriptive `SyntaxError`.

---

## Execution Instructions

### Running Question 1
```bash
# Standard mode
python3 Question-1/regexpert.py < input.txt

# Debug mode (prints DFA transition table to stderr)
python3 Question-1/regexpert.py --debug < input.txt
```

### Running Question 2
```bash
# Valid program or error validation
python3 Question-2/q2.py < program.txt

# Direct terminal input
python3 Question-2/q2.py
move 10; turn left;
# (Press Ctrl+D to send EOF)
```
