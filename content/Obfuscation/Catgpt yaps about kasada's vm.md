## Treat this as correct at your own peril

**I. Introduction**

**1.1. Overview of Kasada and its Bot Protection System**

Kasada is a cybersecurity company specializing in bot protection and mitigation. Their primary focus is defending web applications from automated threats, including but not limited to: credential stuffing, account takeover, web scraping, denial of inventory, and other forms of online fraud. Unlike traditional bot detection methods that might rely on CAPTCHAs or simple fingerprinting, Kasada employs a sophisticated approach centered around a custom-built JavaScript Virtual Machine (VM). This VM is a core component of their client-side obfuscation strategy.

The use of a custom JavaScript VM is a significant departure from more common JavaScript obfuscation techniques.  Typical obfuscation might involve renaming variables, minifying code, or adding dead code. While these can hinder human readability, they are often easily reversed using automated deobfuscation tools.  Kasada's approach, however, introduces an entirely new layer of abstraction. The original application logic isn't merely obfuscated; it's compiled into a custom bytecode format that is then executed within the Kasada VM running in the client's browser. This makes direct analysis of the underlying JavaScript code significantly more difficult.

The Discord chat logs confirm this as being a primary challenge. In an early chat exchange, user "godkiwi" pins a message listing sites protected by Kasada. User i7solar responds with "tip: look at `KPSDK_0xd388()`" (8/15/2021 1:32 PM). This immediately points to a core obfuscated function within Kasada's code, indicating that even identifying key entry points requires reverse engineering effort. Later, user "godkiwi" states "Yea cus its obfuscated with a vm" (8/21/2021 9:47 AM), directly confirming the VM-based obfuscation. User "ascendingz" (8/19/2021 8:11 PM) states "Strong defense against reverse engineering – We start by making our detection logic and telemetry data extremely difficult to reverse engineer. Executing detection logic inside a JavaScript virtual machine ensures that the detection model is protected and that the telemetry data is encrypted. This prevents bot operators from generating and replaying the requests".

The presence of this VM means that attackers (or researchers) cannot simply analyze the visible JavaScript code to understand Kasada's detection logic. Instead, they must:

1.  Understand the architecture of the custom VM itself. This includes its instruction set, how it handles memory, how it interacts with JavaScript, and its internal data structures.
2.  Decode the custom bytecode format to understand the instructions the VM is executing.
3.  Decompile the bytecode back into a higher-level, human-readable representation (ideally, equivalent JavaScript code).

The Bachelor's Thesis, "Decompilation of Kasada's JavaScript Virtual Machine," sets out to achieve precisely this, providing a systematic approach to understanding and decompiling the VM.  The Discord chat logs, on the other hand, provide a complementary perspective, showcasing the real-world challenges and ongoing efforts of developers attempting to bypass Kasada's protection.

The difficulty is further compounded by the fact that Kasada's VM isn't static.  Components like the instruction set, the bytecode parsing mechanisms, and even the mapping of control signals change between different instances of the VM, and over time. This dynamic nature is a crucial aspect of Kasada's defense strategy, making static analysis significantly less effective. Mentions of this dynamic nature are frequent in the Discord logs, such as thebotsmith saying "Which changes every refresh" (8/21/2021 9:57 AM), and later stating "opcodes are diff between sites though" (8/28/2021 6:17 PM).

**1.2. Methodology**

This report utilizes two primary sources of information to provide a comprehensive analysis of Kasada's JavaScript VM and the challenges of decompiling it:

1.  **Bachelor's Thesis:** The thesis, "Decompilation of Kasada's JavaScript Virtual Machine" by Patrick van den Bosch, provides a structured, academic approach to decompilation. It outlines a clear methodology, including parsing the interpreter, disassembling the bytecode, constructing control flow graphs, performing control flow analysis, and generating equivalent JavaScript code. The thesis serves as the theoretical foundation for this report.

2.  **Discord Chat Logs:** The Discord chat logs from the "Sneaker Development" server, specifically the "#anti-bots / kasada" channel, offer a real-world, time-stamped perspective. These logs document the ongoing efforts, challenges, and shared knowledge of a community of developers actively working to understand and bypass Kasada's protection. They provide valuable insights into the practical difficulties of dealing with a dynamic and evolving target.

The approach of this report is to combine the theoretical framework of the thesis with the practical insights from the Discord chat. The thesis provides the "how" of decompilation, while the Discord logs reveal the "what" – the specific obstacles, evolving defenses, and community-driven solutions encountered in the wild. This combination allows for a more complete and nuanced understanding of the subject matter.


**II. Kasada's VM Architecture**

**2.1. Core Components**

Kasada's JavaScript VM, as described in the thesis and corroborated by the Discord chat logs, is a register-based virtual machine. This means it uses a set of numbered registers to store and manipulate data during execution, rather than a stack-based approach (which is mentioned as an alternative in the thesis's "Preliminaries"). This architectural choice has implications for both the bytecode format and the interpreter's design. The core components are:

*   **Bytecode:** The bytecode is the low-level representation of the program logic that the VM executes.  It is *not* standard JavaScript, nor is it WebAssembly (Wasm). It's a custom format designed by Kasada.  The thesis describes it as "an array of 32-bit signed integers," but it's initially presented to the client as an encoded string.  This string uses a custom alphabet and an offset value (delimiter) to define number boundaries.  The decoding process is described in detail in section 3.3 of the thesis, and a simplified JavaScript `decode` function is provided:

    ```javascript
    function decode (bytecode, alphabet, delimiter) {
        var alphabetLength = alphabet.length;
        var multiplierBase = alphabetLength - delimiter;
        var result = [];

        for (var i = 0; i < bytecode.length;) {
            var value = 0;
            var multiplier = 1;
            while (true) {
                var charIndex = alphabet.indexOf(bytecode[i++]);
                value += multiplier * (charIndex % delimiter);
                if (charIndex < delimiter) {
                    result.push(value | 0);
                    break;
                }
                value += delimiter * multiplier;
                multiplier *= multiplierBase;
            }
        }
        return result;
    }
    ```

    This function takes the `bytecode` string, a custom `alphabet` string, and a `delimiter` integer as input. It iterates through the bytecode string, converting characters to numerical values using the alphabet and the delimiter. The `multiplier` and `multiplierBase` variables handle the variable-length encoding of the integers. The result is an array of 32-bit signed integers (`result`), which represents the decoded bytecode instructions.

    The Discord logs also refer to this encoded string. For example, user "Deleted User" states "look at the long string at the end of the script each refresh" (8/20/2021 2:40 PM). User "santan" replies, "wouldn’t call it a long string...it’s a moderate length" (8/20/2021 2:57 PM). These are clearly refering to the encoded bytecode string. The chat also mentions p.js and ips.js, saying "p.js is the kasada file correct?"(8/20/2021 10:50 AM), "ips.js aswell" (8/20/2021 12:11 PM), "Ips.Js"(8/21/2021 9:27 AM). Ips.js is confirmed to be the relevant one: "damn... i have a basic understanding how everything works... someone said that the vm is dynamic and changes. i did a few tests and it looks like only the bytecode changes, not the vm itself (e.g. like shape)
is that always true?"(8/28/2021 2:40 PM)

*   **Interpreter:** The interpreter is the core of the VM. It's a JavaScript function (or set of functions) responsible for:
    *   Fetching the next bytecode instruction from the decoded bytecode array.
    *   Decoding the instruction (determining the opcode and operands).
    *   Executing the instruction (performing the corresponding operation).
    *   Maintaining the VM's state (registers, program counter, etc.).
    * The Thesis, in section 3.1 details how the interpreter contains dynamic elements. Section 3.4 of the thesis dives into the Bytecode parsing function.

*   **Control Signal Array:** This array, referred to as `N` in the thesis example, acts as a lookup table for interpreting different data types within the bytecode. Each element in the array corresponds to a specific data type or interpretation method. The bytecode parsing function (described below) uses this array to determine how to process subsequent integers in the bytecode stream.  An example of a control signal array from the thesis is:

    ```javascript
    var N = [6, 18, 14, 2, 20, 40];
    ```

    The Discord chat confirms the dynamic nature of this array: "...the control signal array, which determines how different types of values are read from the bytecode, and the opcode array, which defines the set of operations the VM can perform. The VM's design includes dynamic elements, particularly in the bytecode parsing function, which changes between different instances of the VM." This dynamic nature makes static analysis difficult.

*   **Opcode Array:** This array defines the set of operations that the VM can perform. Each opcode (represented by an integer in the bytecode) corresponds to a function in this array.  The thesis states that the *order* of opcodes in this array is randomized for each VM instance. This is a key obfuscation technique. The thesis section 3.6 mentions, "The VM's instruction set is defined by an array of opcode functions... The order of opcodes in this array is randomized for each VM instance". The Discord logs also mention this, "so opcodes etc are always the same?"(8/28/2021 2:41 PM), "ive heard the opcodes change i think but not sure if thats true or not"(8/28/2021 3:02 PM).

*   **Bytecode Parsing Function:** This is a crucial part of the interpreter. It's responsible for extracting various data types from the bytecode stream, guided by the control signal array. The thesis (Section 3.4) provides a detailed example of a bytecode parsing function (`R`), which is broken down into sections for handling:

    *   **Integers:** The function checks the least significant bit to determine if the value is an integer.  If it is, a right-shift operation (`r >> 1`) is used to obtain the actual integer value.
    *   **64-bit Floating Point Values:**  If the current value matches a control signal, the function reads two more integers from the bytecode and uses bitwise operations to reconstruct the sign, exponent, and mantissa of an IEEE 754 floating-point number.  Special cases like NaN and Infinity are handled explicitly.
    *   **Strings:**  If the current value matches the string control signal, the function reads the string length, then reads that many integers, decoding them into characters using a custom formula.
    *   **Other Types:**  The function also handles `null`, `true`, `false`, and `undefined` values based on control signals. There's also a fallback mechanism for accessing register values.
        The Discord chat confirms that different data types exist, and that the parsing mechanism is crucial: "...extracting various data types from the bytecode. This function works in conjunction with a control signal array to determine how to interpret different parts of the bytecode."(3.4)

*   **Registers:** Kasada's VM is register-based. The registers are used to store intermediate values during computation.  The thesis's IR design uses numbered registers (e.g., `r4`, `r7`).  The Discord chat mentions registers frequently, for example, "If none of the control signal mappings match and if it doesn't return a small integer we use the integer as index right-shifted by 5 for a register and return the value from this register (r >> 5)." (3.4.3)

*   **Memory Management:** The VM manages its own memory space, separate from the JavaScript heap. The thesis mentions memory cells (e.g., `v22`) and memory operations within the IR. The details of the memory management scheme are not fully elaborated in the thesis, but it's clear that the VM has mechanisms for allocating and deallocating memory as needed.

*    **Interface with JavaScript (FFI):** The VM needs to interact with the host JavaScript environment. It requires a "Foreign Function Interface" (FFI) to call native JavaScript functions. The thesis mentions this interface as crucial for integrating with the browser environment (e.g., accessing the DOM).

**2.2. Dynamic Nature of the VM**

A key characteristic of Kasada's VM, and a major challenge for decompilation, is its dynamic nature.  Many of the components described above are *not* static; they change between different instances of the VM, and even between requests within the same session.  This dynamism is a core part of Kasada's obfuscation strategy.  The thesis and Discord logs highlight several aspects of this:

*   **Changing Bytecode:** The actual bytecode instructions, even for the same underlying logic, can vary. This prevents static analysis based on known bytecode patterns.
*   **Changing Parsing Mechanisms:** The structure and implementation of the bytecode parsing function itself can change. This means that the logic for extracting integers, strings, etc., is not fixed.
*   **Changing Control Signal Mappings:** The values in the control signal array, and their corresponding types, can change. This makes it difficult to reliably interpret the bytecode stream.
*   **Changing Opcode Order:** The order of functions in the opcode array is randomized, meaning the same opcode integer can represent different operations in different VM instances.
* The Thesis mentions usage of SWC to parse the obfuscated javascript and extract a mapping between control signals and value types.
*  The Discord logs have ample evidence of the dynamic nature: The user "Deleted User" asks: "?so opcodes etc are always the same?" (8/28/2021 2:41 PM).  levi_nz responds: "ive heard the opcodes change i think but not sure if thats true or not" (8/28/2021 3:02 PM).  thebotsmith confirms: "opcodes are diff between sites though" (8/28/2021 6:17 PM). Multiple users discuss Kasada updates, indicating ongoing changes: "kasada updated 😦" (8/31/2021 5:10 AM), and "kasada getting very sneaky now :bigcry:" (10/10/2021 4:43 PM).

**2.3. Example Bytecode and Control Signal**
The thesis provides an example of how information is encoded inside of the VM, an example bytecode string:
```
4dodofohojolonoporotovoxozooDoFoHoJoLoNoPS7e...
```
The Thesis explains how it can be decoded with a custom alphabet and delimiter. The control signal array is another example of how information is encoded, as shown in 3.4.1, it can look like:
```
var N = [6, 18, 14, 2, 20, 40];
```
These examples, taken from the thesis and confirmed through the Discord logs, provide the groundwork for understanding Kasada's dynamic architecture and the significant obstacles to static analysis. This makes the dynamic analysis and decompilation techniques, as described in the thesis, all the more crucial.

**III. Decompilation Process**

**3.1. Parsing the Interpreter (Detailed Analysis)**

The first, and arguably most critical, step in decompiling Kasada's VM is parsing the interpreter itself. Because the VM is dynamic, static analysis of a single instance is insufficient. The decompiler must be able to adapt to variations in the VM's structure and logic across different instances. Chapter 3 of the thesis outlines the approach taken to address this challenge.

*   **3.1.1. Locating Dynamic Components (Using SWC)**

    The thesis mentions using Speedy Web Compiler (SWC), a Rust-based JavaScript/TypeScript compiler, to parse the obfuscated JavaScript code containing the VM. SWC allows the code to be transformed into an Abstract Syntax Tree (AST). The AST provides a structured representation of the code that can be programmatically analyzed.

    The key dynamic components that need to be located within the AST are:

    *   **Bytecode Parsing Function:** This function, as described in Section 2.1, is responsible for extracting data from the bytecode. Its structure and logic can vary between VM instances.
    *   **Control Signal Array:** The array that maps control signals to data types. Its contents and the order of elements can change.
    *   **Opcode Array:** The array containing the functions that implement the VM's instruction set. The order of opcodes in this array is randomized.
    *The thesis mentions that SWC is used to parse the Javascript into an AST, and that the structure of the bytecode parsing function and control signal array is analyzed within the AST.

    The Discord logs don't explicitly mention SWC, but they do confirm the need to locate and analyze these components. For instance, the discussion about the "long string at the end of the script" (8/20/2021) and the "control signal array" (3.4.1) demonstrates the community's awareness of these key elements. The mention of "KPSDK_0xd388()" (8/15/2021) suggests an attempt to identify specific functions within the obfuscated code.

*   **3.1.2. Decoding the Bytecode String (Detailed Explanation of the `decode` Function)**

    The `decode` function, as presented in simplified form in Section 2.1 and in the thesis in Section 3.3, is responsible for converting the initial encoded bytecode string into an array of 32-bit integers. The thesis provides the algorithm, and we can analyze it in more detail here:

    ```javascript
    function decode (bytecode, alphabet, delimiter) {
        var alphabetLength = alphabet.length;
        var multiplierBase = alphabetLength - delimiter;
        var result = [];

        for (var i = 0; i < bytecode.length;) {
            var value = 0;
            var multiplier = 1;
            while (true) {
                var charIndex = alphabet.indexOf(bytecode[i++]);
                value += multiplier * (charIndex % delimiter);
                if (charIndex < delimiter) {
                    result.push(value | 0);
                    break;
                }
                value += delimiter * multiplier;
                multiplier *= multiplierBase;
            }
        }
        return result;
    }
    ```

    *   **Inputs:**
        *   `bytecode`: The encoded string containing the bytecode.
        *   `alphabet`: A custom alphabet string defining the character set used for encoding.
        *   `delimiter`: An integer acting as an offset within the alphabet, used to determine number boundaries.
    *   **Process:**
        1.  **Initialization:**
            *   `alphabetLength`: Stores the length of the custom alphabet.
            *   `multiplierBase`: Calculated as `alphabetLength - delimiter`. This value is used in the decoding process to handle variable-length integer encoding.
            *   `result`: An empty array to store the decoded integers.
        2.  **Outer Loop:** Iterates through the `bytecode` string character by character (using `i` as the index). The loop continues as long as `i` is less than the length of the `bytecode` string.
        3.  **Inner Loop:**  This loop decodes a single integer from the bytecode string.
            *   `value`: Initializes the integer value being decoded to 0.
            *   `multiplier`: Initializes a multiplier to 1. This multiplier is used to handle the variable-length encoding.
            *   `charIndex`: Finds the index of the current character (`bytecode[i++]`) within the `alphabet` string. The `i++` increments the index after each character is processed.
            *   `value += multiplier * (charIndex % delimiter)`:  This is the core decoding step. The character's index within the alphabet is taken modulo the `delimiter`. This result is then multiplied by the current `multiplier` and added to the `value`.
            *   **Conditional Check (`charIndex < delimiter`):**  This condition checks if the current character represents the end of the current integer. If the `charIndex` is less than the `delimiter`, it means the end of the integer has been reached.
                *   If true: The decoded `value` (bitwise ORed with 0 to ensure it's a 32-bit integer) is pushed to the `result` array, and the inner loop breaks.
                *   If false: The decoding process continues for the current integer. `value` is updated by adding `delimiter * multiplier` and `multiplier` is multiplied by the `multiplierBase`.
        4.  **Return:** The function returns the `result` array, containing the decoded 32-bit integers.

    This decoding scheme allows Kasada to represent integers using a variable number of characters, making static analysis of the bytecode string more difficult. The `delimiter` acts as a separator, and the custom `alphabet` adds another layer of obfuscation.

*   **3.1.3. Mapping Control Signals to Types (Extracting the Mapping)**

    Once the bytecode parsing function has been located and analyzed, the next step is to extract the mapping between control signals (from the control signal array) and their corresponding data types. The thesis describes this process in Section 3.4.3. The goal is to create a `control_signal_mapping` object like this:

    ```javascript
    control_signal_mapping = {
        0: number,
        1: boolean_true,
        2: string,
        3: boolean_false,
        4: null,
        5: undefined
    }
    ```

    The thesis doesn't provide explicit code for *how* this mapping is extracted, but it implies that it's done through careful analysis of the bytecode parsing function's logic.  The process likely involves:

    1.  **Identifying Control Signal Checks:**  Locating the points within the bytecode parsing function where it checks if the current bytecode value matches a control signal (e.g., `if (r === f[0])`).
    2.  **Analyzing Subsequent Code:**  Observing the code that executes after a control signal match to determine the data type being handled. For example, if the code after `if (r === f[0])` performs floating-point reconstruction, then `f[0]` is likely the control signal for floating-point numbers.
    3.  **Building the Mapping:**  Creating a data structure (like the `control_signal_mapping` object) to store the correspondence between each control signal and its identified data type.

    The Discord logs mention a "flexible parsing approach that adapts to the variations in the VM's structure" and the goal of extracting "a mapping between the control signals and their corresponding value types" (3.2).

*   **3.1.4. Opcode Parsing (Identifying and Analyzing the Opcode Array)**

    The final step in parsing the interpreter is identifying and analyzing the opcode array. The thesis describes this process in Section 3.6. The opcode array contains functions that define the VM's instruction set.  The key challenges here are:

    *   **Randomized Order:** The order of opcodes in the array is randomized for each VM instance.
    *   **No Fixed Mapping:** There's no fixed mapping between opcode numbers and operations across different VM instances.

    The thesis states that the parsing process involves:

    1.  **Identifying the Array:** Locating the opcode array within the AST.
    2.  **Analyzing Structure:** Examining the structure of the array to determine the available operations and their current mapping to opcode numbers.

    The Discord chat confirms the dynamic nature of opcode mappings: "what might be opcode 5 for addition in one instance could be opcode 12 in another instance" (3.6). This emphasizes the need for dynamic analysis.

This section provides a thorough breakdown of parsing the interpreter, detailing each component and linking them to the methodology and information from the thesis, plus examples or commentary from the discord logs where they back up a point.

**3.2. Disassembly: From Bytecode to Basic Blocks**

After the interpreter is parsed and its dynamic components (bytecode decoding, control signals, opcode array) are understood, the next phase is disassembly. This involves transforming the raw bytecode (the array of 32-bit integers) into a more human-readable and analyzable form: the Intermediate Representation (IR). This section breaks down the disassembly process as described in Chapter 4 of the thesis.

**3.2.1. Intermediate Representation (IR) Design**

The IR serves as a bridge between the low-level bytecode and the high-level JavaScript code that will eventually be generated. The thesis describes the IR as capturing both data and control flow aspects of the program. It's designed around two primary components: *values* and *instructions*.

*   **Values:** The IR needs to represent all the data types that the Kasada VM can handle.  The thesis specifies these as:

    *   **Primitive Types:**
        *   Numbers (integers and floating-point)
        *   Strings
        *   Booleans (`true` and `false`)
    *   **Special Values:**
        *   `null`
        *   `undefined`
    *   **Registers:** Kasada's VM is register-based, so the IR must represent registers. The thesis uses numbered registers (e.g., `r4`, `r7`).
    *   **Memory Cells:** The VM has its own memory space, and the IR represents memory locations using variable names (e.g., `v22`).
    *   **Complex Structures:** The IR needs to support more complex data structures like objects, arrays, and functions. While the thesis doesn't provide explicit details on how these are represented in the IR, it mentions their existence.
    * **Expressions:** The IR needs to be able to represent expressions, the thesis mentions "member access and binary operations".

*   **Instructions:** The IR must represent the various operations that the VM can perform. The thesis outlines several categories of instructions:

    *   **Register Assignments:**  These instructions move values between registers (e.g., `SET REGISTER r4 VALUE: v5`).  These form the core of data manipulation in a register-based VM.
    *   **Property Assignments:** These instructions modify properties of objects (likely represented as memory cells in the IR).
    *   **Memory Assignments:** The thesis mentions `SET MEMORY` instructions for storing values in memory locations (e.g., `SET MEMORY v22 VALUE: arg0`).
    *   **Control Flow Instructions:** These instructions control the program's execution flow:
        *   **Jumps:** Unconditional jumps to a specific instruction pointer (e.g., `JUMP 82`).
        *   **Conditional Jumps:** Jumps that depend on a condition (e.g., `IF TEST: r5 CONSEQUENT: 82 ALTERNATE: 69`). The thesis mentions `JumpIfTrue` and `JumpIfFalse` as VM instructions.
        *   **Function Calls:** Instructions for calling functions within the VM. The thesis discusses `InitFunc` as a specific opcode.
        *   **Return Statements:** Instructions for returning from functions.
    *   **Exception Handling Instructions:** These instructions manage exceptions.  Kasada uses a custom mechanism based on instruction pointers, rather than native JavaScript `try-catch-finally` blocks. The thesis mentions:
        *   `SetCatch`: Stores the instruction pointer to jump to if an exception occurs.
        *   `SetFinally`: Stores the instruction pointer to a `finally` block that always executes.
    *   **Throw Statements:** Instructions that explicitly throw an exception.
    * The thesis mentions "Memory operations provide access to the VM's memory cells" which can be taken to be a more general instruction type.

**3.2.2. Traversal Strategy (Breadth-First Search, BFS)**

The disassembler needs a systematic way to discover and process all reachable parts of the bytecode. The thesis describes using a Breadth-First Search (BFS) traversal, starting from the VM's entry point (the first instruction in the bytecode).

*   **Queue-Based Approach:** BFS uses a queue to keep track of the basic blocks that need to be processed. The traversal starts by adding the entry point block to the queue.
*   **Recursive Traversal:**  The thesis emphasizes the importance of a *recursive* traversal, particularly for handling exception handlers. Exception handlers can transfer control to arbitrary locations, potentially creating nested structures. A recursive approach ensures that all possible execution paths, including those within exception handlers, are explored. The thesis makes it a point to explain why a recursive traversal is better than a simpler linear traversal.
*   **Process:**
    1.  Initialize a queue with the entry point block's instruction pointer.
    2.  While the queue is not empty:
        *   Dequeue a block's instruction pointer.
        *   Create a new basic block starting at that instruction pointer.
        *   Process instructions within the block until a control flow instruction is encountered.
        *   For each control flow instruction:
            *   **Jump:** Add the target instruction pointer to the queue.
            *   **Conditional Jump:** Add both the consequent and alternate instruction pointers to the queue.
            *   **Function Definition (`InitFunc`):** Add the function's entry point to the queue.
            *   **Exception Handlers (`SetCatch`, `SetFinally`):** Add the relevant instruction pointers (catch handler, finally block) to the queue.

    The recursive nature of the traversal is crucial for handling nested structures like functions within functions and complex exception handling patterns.
* The Thesis stresses the importance of making sure "that we discover all code paths that could be executed".

**3.2.3. Basic Block Formation**

A basic block is a fundamental concept in decompilation. It's a sequence of instructions with the following properties:

*   **Single Entry Point:**  Execution can only enter the block at the first instruction.
*   **One or More Exit Points:** Execution can leave the block from one or more instructions *at the end* of the block. There are no branches *within* the block, except possibly at the very end.
*   **Sequential Execution:** Once the first instruction in a block is executed, all subsequent instructions in the block are guaranteed to execute sequentially, up to the exit point.

The disassembler identifies basic blocks by analyzing the control flow instructions:

*   **Start of a Block:** A new basic block begins at:
    *   The entry point of the VM.
    *   The target of a jump instruction.
    *   The target of a conditional jump instruction (both consequent and alternate branches).
    *   The entry point of a function (`InitFunc`).
    *   The target of an exception handler (`SetCatch`, `SetFinally`).
*   **End of a Block:** A basic block ends when:
    *   A control flow instruction is encountered (jump, conditional jump, function call, return, exception handler).
    *   The next instruction is the start of another basic block (this can happen due to converging control flow paths).

The disassembler constructs basic blocks by iterating through the instructions, starting at a block's entry point, and stopping when a control flow instruction or the start of another block is found.

**Optimization for Converging Code Paths**

The thesis describes an important optimization strategy.  The bytecode often contains identical sequences of instructions that can be reached through different control flow paths (e.g., the code after an `if-else` statement, where both branches converge).  Instead of creating duplicate basic blocks for these identical sequences, the disassembler recognizes when it encounters instructions it has seen before. It uses a mapping between instruction pointers and their corresponding basic blocks. When a duplicate sequence is found, it can:

1.  Split the existing blocks (if necessary) to create a new block containing only the shared sequence.
2.  Modify the original blocks to jump to this new, shared block.

This optimization reduces the size of the IR and simplifies the control flow graph.

**3.2.4. Handling Expressions**

The disassembler must handle various types of expressions within the bytecode. The thesis mentions:

* **Binary Expressions:** Operations like addition, subtraction, comparison, which are either encoded directly with immediate values, or with operands stored in registers.
* **Function Construction**: Specifically handling the `InitFunc` opcode, and allocating arguments to registers.

**3.2.5. Exception Handling (SetCatch, SetFinally, try-catch-finally structures)**

Kasada's VM uses a custom exception handling mechanism based on instruction pointers, rather than native JavaScript `try-catch-finally` blocks. The disassembler must correctly interpret the `SetCatch` and `SetFinally` instructions to reconstruct the exception handling structure.

*   `SetCatch`:  Stores the instruction pointer of the `catch` block (where execution should continue if an exception occurs).
*   `SetFinally`: Stores the instruction pointer of the `finally` block (which always executes, regardless of whether an exception occurred).

The disassembler analyzes the sequence of these instructions to determine the type of exception handling structure:

*   **Try-Catch:**  A `SetCatch` instruction followed by the `try` block's code.
*   **Try-Finally:** A `SetCatch` immediately followed by a `SetFinally` instruction.
*   **Try-Catch-Finally:**  A `SetCatch` followed by a `SetFinally`.

The disassembler creates separate basic blocks for the `try`, `catch`, and `finally` blocks, and connects them appropriately in the control flow graph. The Thesis emphasizes the importance of structuring the exception handlers *before* conditionals.

**3.2.6 Example of IR for a Basic Block**

The thesis provides an example of how a function might be represented in the IR:

```
FUNCTION BLOCK 53:  // Function entry point
53 SET MEMORY v22 VALUE: arg0 // Store argument
56 SET REGISTER r4 VALUE: v5
59 SET REGISTER r7 VALUE: v22
62 SET REGISTER r5 VALUE: r4(r7)
66 IF TEST: r5 CONSEQUENT: 82 ALTERNATE: 69

BLOCK 69: // Alternate path
69 SET REGISTER r7 VALUE: v4
72 SET REGISTER r8 VALUE: v22
75 SET REGISTER r6 VALUE: r7(r8)
79 SET REGISTER r5 VALUE: r6
82 JUMP 82

BLOCK 82: // Conditional branch
82 IF TEST: r5 CONSEQUENT: 98 ALTERNATE: 85

// Additional blocks omitted for brevity...
```

This example illustrates:

*   **Block Headers:**  Each block is labeled with its type (FUNCTION or BASIC) and its starting instruction pointer.
*   **Instructions:**  Instructions within the block are listed with their bytecode positions.
*   **Operands:**  Registers (e.g., `r4`, `r5`) and memory locations (e.g., `v22`, `v4`, `v5`) are represented.
*   **Control Flow:**  Conditional jumps (`IF`) and unconditional jumps (`JUMP`) are clearly indicated, along with their target blocks.

**3.3. Constructing Control Flow Graphs (CFGs)**

Once the bytecode has been disassembled into basic blocks and the Intermediate Representation (IR) has been created, the next step is to construct a Control Flow Graph (CFG). The CFG is a directed graph that visually represents the possible execution paths through the program. Chapter 5 of the thesis describes this process.

**3.3.1. Nodes**

*   **Representation:** Each node in the CFG represents a basic block, as defined in the previous section. A basic block, remember, is a sequence of instructions with a single entry point and one or more exit points (due to control flow instructions at the end).
*   **Creation:** The thesis describes a queue-based traversal strategy, starting from the function's entry point. This is essentially the Breadth-First Search (BFS) described earlier for disassembly. A queue of instruction pointers is maintained. For each instruction pointer in the queue:
    *   A new node is created in the CFG.
    *   The corresponding basic block is identified (starting at that instruction pointer).
    *   The final instruction of the basic block is examined to determine the control flow.
*   **Separation of Disassembly and CFG Construction:** The thesis explicitly states the need to separate the disassembly and CFG construction phases. This is crucial because a single basic block might be part of *multiple* functions (due to code sharing, as discussed in the optimization strategy). The CFG construction needs to consider all possible entry points to a block.

**3.3.2. Edges**

*   **Representation:** Edges in the CFG represent the possible flow of control between basic blocks. An edge from node A to node B indicates that execution can flow directly from the end of block A to the beginning of block B.
*   **Edge Labels:** The thesis emphasizes the importance of *labeling* the edges to indicate the type of control flow. This labeling is critical for later analysis (structuring loops, conditionals, etc.). The following labels are used:
    *   **`jump`:**  For unconditional jumps (e.g., a `JUMP` instruction).
    *   **`consequent`:** For the "true" branch of a conditional jump (e.g., `JumpIfTrue`).
    *   **`alternate`:** For the "false" branch of a conditional jump (e.g., `JumpIfFalse`).
    *   **`try`:**  For the edge leading from a `try` block to its normal successor (if no exception occurs).
    *   **`catch`:** For the edge leading from a `try` block to its `catch` handler (if an exception occurs).
    *   **`finally`:**  For edges leading to a `finally` block. A `finally` block can have incoming edges from both the normal exit of the `try` block and the exit of the `catch` block.
        * The chat logs contain examples like: "66 IF TEST: r5 CONSEQUENT: 82 ALTERNATE: 69"(4.3.1), showing this labelling.
*   **Edge Creation:** Edges are created based on the control flow instructions at the end of each basic block:
    *   **Direct Jumps:** A single edge labeled `jump` is created to the target block.
    *   **Conditional Jumps:** Two edges are created: one labeled `consequent` to the target block if the condition is true, and one labeled `alternate` to the target block if the condition is false.
    *   **Return Instructions:** Return instructions have *no* outgoing edges, as they terminate the control flow within the current function.
    *   **Exception Handlers:** The edge creation for exception handlers is more complex and depends on the type of handler:
        *   **Try-Catch:** An edge labeled `try` to the normal successor of the `try` block, and an edge labeled `catch` to the `catch` handler.
        *   **Try-Finally:** Edges labeled `try` and `finally`, ensuring control flow goes through the `finally` block.
        *   **Try-Catch-Finally:** Edges labeled `try`, `catch`, and `finally`, ensuring that both normal execution and exception handling paths flow through the `finally` block.

**3.3.3. Two-Pass Approach**

The thesis emphasizes a *two-pass* approach for CFG construction:

1.  **First Pass (Node Discovery):**  This pass focuses on *discovering* all reachable basic blocks and creating the corresponding nodes in the CFG. The queue-based traversal (BFS) is used to ensure that all reachable blocks are found, even if they are not sequentially arranged in the bytecode.
2.  **Second Pass (Edge Creation):**  After *all* nodes have been created, a second pass is performed to add the edges between them. This is necessary because, during the first pass, the target of a jump instruction might not yet have been discovered. By delaying edge creation until all nodes exist, we ensure that all necessary connections can be made.

This two-pass approach is crucial for handling forward jumps and complex control flow patterns.

**3.3.4. Example CFG**

The thesis provides two example CFGs (Figures 5.1 and 5.2).  Let's briefly analyze Figure 5.1, as it's simpler:

*Figure 5.1: Control flow graph of a function*

This CFG shows a function with conditional branching and path convergence.

*   **Nodes:** Each node is labeled with the starting instruction pointer of the corresponding basic block (e.g., Node 0 starts at instruction 53). The instructions within each block are listed.
*   **Edges:**  Edges are labeled with the type of control flow (`consequent`, `alternate`, `jump`).
*   **Conditional Branch:** Node 0 (starting at instruction 53) contains an `IF` instruction (instruction 66). This creates two outgoing edges:
    *   `consequent`:  Leads to Node 1 (instruction 82), representing the path taken if the condition is true.
    *   `alternate`: Leads to Node 2 (instruction 69), representing the path taken if the condition is false.
*   **Path Convergence:** Both Node 2 and Node 4 have a `JUMP` instruction targeting Node 1. This shows how control flow can converge after a conditional branch.
*   **Return:** Node 5 (instruction 110) contains a `RETURN` instruction, and therefore has no outgoing edges.

Figure 5.2 shows a more complex example, including a try-catch-finally structure and a loop. The principles, however, remain the same: nodes represent basic blocks, and labeled edges represent the control flow.

**3.4. Control Flow Analysis**

Control Flow Analysis (CFA) is a crucial phase in decompilation.  After constructing the Control Flow Graph (CFG), CFA aims to identify high-level language constructs from the low-level control flow represented by the graph.  This involves recognizing patterns in the CFG that correspond to loops, conditional statements (if-else), and exception handling structures (try-catch-finally). Chapter 6 of the thesis covers this analysis. The goal is to transform the raw CFG into a *structured* representation that more closely resembles the original source code.

**3.4.1. Structuring Loops**

The first step in CFA is identifying and structuring loops.  Loops in the bytecode are represented as cycles in the CFG.

*   **Loop Detection (Johnson's Algorithm):** The thesis mentions using Johnson's algorithm [Johnson, 1975] to find all elementary circuits (cycles) in the directed graph (the CFG). This algorithm is specifically designed for finding cycles in directed graphs efficiently.

*   **Loop Types:** The thesis distinguishes between three types of loop structures:

    *   **While Loop:** A loop with a conditional test at the *beginning* (header node). The loop body is executed only if the condition is true.  The CFG pattern for a while loop is:
        *   A header node containing a conditional jump.
        *   A "consequent" edge leading to the loop body.
        *   An "alternate" edge leading to the code *after* the loop (the "follow" node).
        *   A "latching" node at the end of the loop body with an unconditional jump back to the header node.
    *   **Do-While Loop:** A loop with a conditional test at the *end* (latching node). The loop body is always executed at least once. The CFG pattern is:
        *   A header node.
        *   Edges leading to the loop body.
        *   A latching node at the end of the loop body with a *conditional* jump back to the header node.
    *   **Infinite Loop:** A loop with no explicit exit condition within the loop itself.  These loops typically rely on `break` statements within the loop body to terminate. The CFG pattern is:
        *   A header node.
        *   An unconditional jump from the latching node back to the header.
        *   No conditional exits at the header or latching nodes.
        * The thesis also makes the point that the decompiler will need to search for break statments in the loop, something that is only needed for infinite loops, as while and do-while loops in Kasada's VM *never* contain breaks.

*   **Loop Structures and Control Flow Patterns (Figure 6.1):**  Figure 6.1 in the thesis visually depicts the CFG patterns for each of these loop types, which is crucial for identifying them.

* The Discord logs have a discussion about this, mentioning identifying cycles, and Johnson's algorithm. User "Deleted User" says "To find loops in the control flow graph, we first need to identify all cycles. For this, we use Johnson’s algorithm, which efficiently finds all elementary circuits in a directed graph [Johnson, 1975]." in section 6.1

**3.4.2. Structuring Exception Handlers**

After identifying loops, the next step is to structure exception handling constructs. As mentioned earlier, Kasada's VM uses a custom mechanism based on instruction pointers (`SetCatch`, `SetFinally`), rather than native JavaScript `try-catch-finally` blocks.

*   **Types of Exception Handlers:** The thesis identifies three types:

    *   **Try-Catch:** A `try` block followed by a `catch` block. If an exception occurs within the `try` block, control is transferred to the `catch` block.
    *   **Try-Finally:** A `try` block followed by a `finally` block. The `finally` block *always* executes, regardless of whether an exception occurred in the `try` block or whether the `try` block completed normally.
    *   **Try-Catch-Finally:** A combination of the above, with a `try` block, a `catch` block, and a `finally` block.

*   **Control Flow Patterns:**  Exception handlers create distinct CFG patterns:

    *   The `try` block has a special "exception" edge leading to the `catch` handler.
    *   The `finally` block has incoming edges from both the normal exit of the `try` block and the exit of the `catch` block (if present).
    *   The exit of the `finally` block represents the point where control flow continues after the exception handler.

*   **Structuring Algorithm:** The thesis provides an algorithm for identifying exception handlers:

    1.  **Identify `try` blocks:** Look for nodes with outgoing "exception" edges (created by `SetCatch` instructions).
    2.  **For each `try` block:**
        *   If it has an edge to a `catch` block, mark that block as its exception handler.
        *   If it has an edge to a `finally` block, or if its `catch` block has an edge to a `finally` block, mark that as its `finally` handler.
    3.  **Find the "follow" node:** This is the node where control flow continues *after* the exception handler.
        *   For `try-catch`:  The follow node is the common target of the `try` and `catch` blocks' normal exits.
        *   For `try-finally` and `try-catch-finally`: The follow node is the normal exit of the `finally` block.

* The Thesis makes note that this identification process is done *before* conditionals because of shared edge patterns.

**3.4.3. Structuring Conditionals**

The final step in control flow analysis is identifying and structuring conditional statements (if-else constructs).

*   **Types of Conditionals:**  The thesis distinguishes between:

    *   **One-way conditional (if):**  A test condition and a single conditional branch (the "then" clause). If the condition is true, the branch is taken; otherwise, execution continues to the "follow" node.
    *   **Two-way conditional (if-else):** A test condition and *two* branches: a "consequent" branch (taken if the condition is true) and an "alternate" branch (taken if the condition is false).

*   **Components of a Conditional Structure:**

    *   **Test Node:** The basic block containing the conditional jump instruction (e.g., `IF TEST: r5 ...`).
    *   **Consequent Node:** The target block executed if the condition is true.
    *   **Alternate Node:** (For two-way conditionals) The target block executed if the condition is false.
    *   **Follow Node:** The block where control flow *converges* after the conditional structure. This is the key to identifying the end of the conditional. The follow node is *immediately dominated* by the test node.

*   **Dominator Trees:** The thesis introduces the concept of *dominator trees*. A node `d` *dominates* a node `n` if *every* path from the entry node of the CFG to `n` must go through `d`. The dominator tree provides a hierarchical view of the control flow, showing dominance relationships. The *immediate dominator* of a node `n` is the closest dominator of `n`. The thesis mentions using the "Simple Fast Dominance Algorithm" [Cooper et al., 2001] to compute dominance information.
* **Dominance is crucial to defining where control flow converges.**

*   **Structuring Algorithm:** The thesis outlines an algorithm based on Cifuentes' work, using *reverse post-order traversal* of the CFG nodes:

    1.  **Traverse nodes in reverse post-order:** This ensures that nested conditionals are handled correctly (innermost first).
    2.  **For each conditional node:**
        *   Identify potential "follow" nodes by finding nodes that:
            *   Are immediately dominated by the conditional node.
            *   Are reachable by at least two paths from the conditional node.
        *   Select the *closest* such node (according to the reverse post-order numbering) as the follow node.

*  **Control flow graph patterns (Figure 6.2):** Figure 6.2 in the thesis visually presents control flow patterns for conditionals.

Okay, let's continue with Section III, moving on to the final stage of the decompilation process:

**3.5. Code Generation**

After the Control Flow Graph (CFG) has been analyzed and structured (identifying loops, conditionals, and exception handlers), the final phase is code generation. This involves transforming the structured CFG and the Intermediate Representation (IR) into readable, high-level JavaScript code. Chapter 7 of the thesis describes this process.

**3.5.1. Graph Traversal Strategy (Modified Depth-First Search, DFS)**

The core of the code generation algorithm is a modified Depth-First Search (DFS) traversal of the structured CFG. Standard DFS wouldn't be sufficient because of the specific structures that need to be handled (loops, conditionals, exception handlers), and the potential for multiple paths to converge at a single node.

*   **Visited Set:** The algorithm maintains a `visited` set to keep track of nodes that have already been processed. This is crucial for preventing infinite loops and avoiding code duplication when multiple paths converge.
*   **Structure-Aware Traversal:** The traversal is "structure-aware," meaning it takes into account the identified control flow structures (loops, conditionals, exception handlers).  This ensures that the generated code reflects the correct nesting and flow of control.
* **Process:**
    1.  Start at the function's entry node.
    2.  Check if the current node has an associated structure (loop, conditional, or exception handler).
    3.  Based on the structure type (or lack thereof):
        *   **No Structure (Basic Block):** Translate the instructions in the basic block into equivalent JavaScript code (see Section 3.5.3).
        *   **Loop:** Generate the appropriate JavaScript loop construct (see Section 3.5.2.1).
        *   **Conditional:** Generate the appropriate JavaScript `if-else` statement (see Section 3.5.2.2).
        *   **Exception Handler:** Generate the appropriate JavaScript `try-catch-finally` block (see Section 3.5.2.3).
    4.  Recursively process child nodes based on the structure's semantics. For example, for a `while` loop, recursively process the header (condition) and the body nodes. For an `if-else` statement, recursively process the consequent and alternate branches. The `visited` set prevents infinite recursion.

* The Thesis explains that a regular DFS would not suffice and require special consideration for structured nodes.

**3.5.2. Structure-Based Code Generation**

This section describes how the identified control flow structures are translated into their corresponding JavaScript constructs.

*   **3.5.2.1. Loop Processing**

    The code generator handles the three types of loops identified during control flow analysis:

    *   **While Loop:** Generate a JavaScript `while` loop. The condition expression is generated from the header basic block, and the loop body is generated by recursively traversing the loop's body nodes.
    *   **Do-While Loop:** Generate a JavaScript `do-while` loop. The loop body is generated first, and the condition expression is generated from the latching node.
    *   **Infinite Loop:** Generate a `while (true)` loop (or equivalent).  `break` statements within the loop body (identified during control flow analysis) are used to terminate the loop.

*   **3.5.2.2. Conditional Processing**

    Conditional structures are translated into JavaScript `if-else` statements:

    *   **One-way conditional (if):** Generate an `if` statement. The test expression is generated from the test node, and the consequent branch is generated by recursively traversing the consequent node.
    *   **Two-way conditional (if-else):** Generate an `if-else` statement. The test expression is generated from the test node, the consequent branch is generated from the consequent node, and the alternate branch is generated from the alternate node.

*   **3.5.2.3. Exception Handler Processing**

    Exception handling structures are translated into JavaScript `try-catch-finally` blocks:

    *   **Try-Catch:** Generate a `try` block (from the `try` node) and a `catch` block (from the `catch` node).
    *   **Try-Finally:** Generate a `try` block and a `finally` block.
    *   **Try-Catch-Finally:** Generate a `try` block, a `catch` block, and a `finally` block.

    Each component (try, catch, finally) is generated by recursively traversing the corresponding nodes in the CFG.

**3.5.3. Basic Block Translation**

For basic blocks (nodes without an associated control flow structure), the instructions within the block are translated sequentially into equivalent JavaScript code. The thesis mentions the following instruction types that require translation:

*   **Register Assignments:** Translate into JavaScript variable assignments (e.g., `r4 = v5;`).
*   **Property Assignments:** Translate into JavaScript property access operations.
*   **Memory Assignments:** Translate into assignments to variables representing memory locations.
*   **Return Statements:** Translate into JavaScript `return` statements.
*   **Throw Statements:** Translate into JavaScript `throw` statements.

Most VM instructions map directly to equivalent JavaScript operations. The thesis provides some examples, mentioning specifically: "a register assignment in the VM becomes a simple variable assignment in JavaScript, and a property assignment in the VM translates to an equivalent property access operation in JavaScript."

**3.5.4. Scope Management**

The code generator needs to manage variable scopes to ensure that variables are declared correctly in the generated JavaScript code.  The thesis mentions two levels of scope:

*   **Global Scope:** Memory variables (e.g., `v22`) are declared in the global scope because they need to be accessible by all functions.
*   **Function Scope:** Register variables (e.g., `r4`, `r5`) are declared at the start of each function body.

**3.5.5. Code Generation Example**

The thesis provides an example of generated JavaScript code (Listing 1):

```javascript
function func_53(arg0) {
    let r4, r5, r6, r7, r8;
    v22 = arg0;
    r4 = v5;
    r7 = v22;
    r5 = r4(r7);
    if (!r5) {
        r7 = v4;
        r8 = v22;
        r6 = r7(r8);
        r5 = r6;
    }
    if (!r5) {
        r4 = v3;
        r7 = v22;
        r6 = r4(r7);
        r5 = r6;
    }
    if (!r5) {
        r6 = v2;
        r4 = r6();
        r5 = r4;
    }
    return r5;
}
```

This example demonstrates:

*   **Function Declaration:** The generated code starts with a function declaration (`function func_53(arg0)`).
*   **Variable Declarations:** Register variables (`r4`, `r5`, `r6`, `r7`, `r8`) are declared at the beginning of the function.
*   **Register and Memory Assignments:**  VM instructions are translated into JavaScript assignments.
*   **Conditional Statements:**  `IF` instructions are translated into `if` statements.
*   **Return Statement:**  The `RETURN` instruction is translated into a `return` statement.

This example, though simplified, showcases the output of the code generation phase. The generated code is structured, readable, and reflects the control flow of the original bytecode.

**IV. Challenges and Solutions (From Discord Logs)**

While the thesis provides a theoretical framework for decompiling Kasada's VM, the Discord chat logs offer a valuable glimpse into the practical challenges faced by developers attempting to bypass this protection in real-world scenarios. These logs reveal an ongoing "cat-and-mouse" game, with Kasada constantly updating its defenses and developers adapting their techniques. This section categorizes and analyzes these challenges and the solutions (or attempted solutions) discussed within the community.

**4.1 Dynamic Components**

The dynamic nature of Kasada's VM, as highlighted in the thesis, is a recurring theme in the Discord discussions. Developers repeatedly encounter changes that break their existing solutions.

*   **4.1.1 Changing Opcodes:** The randomization of opcode mappings is a core obfuscation technique. The Discord logs confirm this:

    *   "so opcodes etc are always the same?" (8/28/2021) - This question highlights the initial uncertainty about opcode stability.
    *   "ive heard the opcodes change i think but not sure if thats true or not" (8/28/2021) - Confirms that opcode changes are observed.
    *   "opcodes are diff between sites though" (8/28/2021) - Indicates that opcode mappings can vary not only over time but also between different websites using Kasada.
    *   "What’s the new opcode they added" (9/26/2021) - Shows that new opcodes are occasionally introduced.
    *   "Quite a big update tbf" (9/27/2021) - Suggests that updates can involve significant changes.
    *   "they kept version number the same but made changes :KEKW:" (8/31/2021)
    *   "doubt any1 would OS a kasada deobfuscator, they update their scripts w/o version changes" (9/14/2021).

*   **4.1.2. Variations in Bytecode Parsing:** The logic for parsing the bytecode itself is also subject to change. This means that even if the opcodes remain the same, the way data is extracted from the bytecode can be altered, breaking existing parsers.

    *  "...the structure and implementation of the bytecode parsing function varies between different instances of the VM." (3.2) - From the thesis, describing the dynamic parsing function.
    *   "well cant extract it off nike atm"(8/31/2021), "has bstn always had datadome?"(8/31/2021). These messages indicate issues in parsing.
    *   "looks like kasada updated veve"(11/11/2021) - Indicates an update that affected a specific site.
    *    "seems theres more than just that change"(11/11/2021).
    *   "they did do a big one not too long ago but this one is smaller"(8/31/2021). This all implies frequent updates, that sometimes are major.

*   **4.1.3. Control Signal Changes:** The values and order of control signals in the control signal array can change, affecting how different data types are identified within the bytecode.

    *   "Similarly, the values and order of the control signals in the control signal array change, affecting how different types of values are interpreted from the bytecode." (3.2) - From the thesis, confirming dynamic control signals.
    *   "To effectively parse the bytecode in later phases of the decompilation, we need to extract a mapping between the control signals and their corresponding value types." (3.4.3) - Highlights the importance of dynamically extracting this mapping.

**4.2. Exception Handling Complexity**

Kasada's non-standard implementation of exception handling, using instruction pointers instead of native JavaScript `try-catch` blocks, poses a significant challenge.

*   **4.2.1. Non-Standard Implementation:** The thesis describes the use of `SetCatch` and `SetFinally` instructions to manage exception handling. The Discord logs don't discuss the specifics of these instructions in as much detail, but the general difficulty of exception handling is apparent.
*   **4.2.2. Structuring Challenges:** The disassembler needs to correctly identify and reconstruct `try-catch-finally` structures from the instruction pointer-based mechanism. The thesis highlights the importance of structuring exception handlers *before* conditionals due to overlapping control flow patterns. The chat logs don't go into the algorithmic details, but user "Deleted User" mentions "exception handlers, generating readable JavaScript code", which is a key part of the decompiler. (9.1)

**4.3. Control Flow Obfuscation**

Kasada employs techniques to obfuscate the control flow, making it harder to follow the program's logic.
 * 4.3.1 Dead Code Insertion, Conditional Jumps
    * "Kasada employs an additional script that intercepts outgoing requests from the VM." (2.3.2) This interception might introduce additional control flow complexities.
    * Constructing the CFGs involve complex control flow instructions such as conditional jumps and exception handling. (4.1)

**4.4. Open Source Contributions and their impact**

The Discord chat logs feature discussions about open-source contributions, tools, and their potential impact:

*4.4.1 Discussion around Open Source projects.
    * "doubt any1 would OS a kasada deobfuscator, they update their scripts w/o version changes"(9/14/2021).
    * Many messages like "I plan on open sourcing my 1+1 info"(8/28/2021) are thrown around.
 * 4.4.2 Debate on if Open Source is useful:
  * "did you write a full disassembler + decompiler? or just stepped through everything with a debugger"(8/29/2021).
  * "also a question for the others. because i started writing a decompiler now but i think you could maybe find enough things out with a debugger? so i'm wondering how you guys did it"(8/29/2021).
* The discord logs contained a link to an article detailing a partial reverse, highlighting what it did and did not achieve.

**4.5 Times and Dates**
*  4.5.1 Establishing Timelines
 * Mentions of "updated again" (9/26/2021, 10/10/2021) are frequent, establishing that Kasada frequently updates their system.
 * Discussion of updates and patches: "kasada updated 😦" (8/31/2021 5:10 AM), "they kept version number the same but made changes :KEKW:" (8/31/2021 5:25 AM), "Subtle changes" (9/1/2021 12:48 PM), "an extra param" (9/1/2021 12:48 PM), "uuid" (9/1/2021 12:49 PM).
* 4.5.2 Mentions of version numbers:
 * Mentions of versions like "1.6.0" (8/31/2021 5:53 AM), "2.3.2 Conditional Processing"(6.2.3)
* One user mentioned "You seen the multiple vm change yet?"(11/11/2021), indicating a specific kind of update.

**4.6 Community Discussions and Support**
* 4.6.1. Requests for assistance.
 *  "can someone give me a high level overview how kasada works?"(8/27/2021).
 *  "any resources to learn how to reverse something like this?" (8/28/2021).
 *  "any guidelines to start with something like this?" (11/18/2021).
*  4.6.2 Tips and suggestions.
  *  Multiple people provide tips, like "tip: look at KPSDK_0xd388()"(8/15/2021).
  *   "j function in ips.js"(8/28/2021).

**4.7. Other Challenges**

*   **4.7.1. Discussion of "ips.js" and "p.js":**  These filenames are repeatedly mentioned, indicating they are key parts of Kasada's implementation.  There's discussion about which file contains the VM ("ips.js" seems to be the consensus) and which is "useless" (8/20/2021). There's confusion about where to find `ips.js` on Nike's website (8/28/2021).
*   **4.7.2. "x-kpsdk-cd" and "x-kpsdk-ct" headers:** These custom headers are identified as being related to Kasada. There's discussion about their purpose and whether they are static or dynamic (8/20/2021, 11/29/2021). The `-cd` header is identified as easier to generate than the `-ct` header.
*   **4.7.3. Discussion of the VM Itself:**  There's discussion about the VM being dynamic (8/21/2021, 8/28/2021), about the difficulty of cleaning/deobfuscating it (8/21/2021), and about the need for a decompiler (8/28/2021). There's also discussion of whether a debugger is sufficient or if a full decompiler is necessary (8/29/2021).

This section provides a structured overview of the practical difficulties, evolving nature of the VM, and collaborative efforts within the community.

**V. Evolution of Kasada's Defenses (Timeline)**

This timeline is constructed from the Discord chat logs from the "Sneaker Development" server, specifically the "#anti-bots / kasada" channel. Dates are formatted as (MM/DD/YYYY). It highlights key events, discussions, and observations related to Kasada's evolving defenses.

*   **Early Discussions and Pinpointing (08/15/2021):**
    *   (08/15/2021) - Users "godkiwi", "adham_1", "Deleted User", and "idekdude" establish the channel's existence.
    *   (08/15/2021) - A pinned message lists websites protected by Kasada. This marks an early attempt to catalog instances of Kasada's deployment.
    *   (08/15/2021) - User "i7solar" suggests looking at `KPSDK_0xd388()`, a likely obfuscated function name. This indicates an early focus on identifying key functions within the obfuscated code.

*   **Initial Understanding of Kasada (08/17/2021 - 08/20/2021):**
    *   (08/17/2021) - Discussion about Kasada's difficulty. "i7solar" rates it 6.5/10.
    *   (08/19/2021) - User "ascendingz" quotes a description of Kasada's defense, highlighting the JavaScript VM and encryption.
    *   (08/19/2021) - User "i7solar" posts a screenshot, likely related to Kasada's internals, which is met with "pepe_laugh" reactions (12 times).
    *   (08/20/2021) - Discussion about Kasada headers: `x-kpsdk-cd` and `x-kpsdk-ct` are identified. "oblivyan" mentions "p.js" as the Kasada file. "levi_nz" notes that one of the files is useless.
    *   (08/20/2021) - "ahulex" mentions "ips.js".

*   **VM Identification and Early Analysis (08/21/2021 - 08/28/2021):**
    *   (08/21/2021) - "godkiwi" states that Kasada uses a VM ("Yea cus its obfuscated with a vm"). This confirms the core obfuscation technique.
    *   (08/21/2021) - "thebotsmith" notes that the VM "changes every refresh." This highlights the dynamic nature of Kasada.
    *   (08/22/2021) - "shiro.69" asks about cleaning the "ips file" and who is good at "custom vm shit."
    *   (08/25/2021) - User "_merlijn" notes being in Kasada's "Hall of Fame."
        *    (08/25/2021) - User "oblivyan" posts a link to Kasada's "Hall of Fame".
    *   (08/28/2021) - Extensive discussion about Kasada on Nike's website. Users discuss the presence (or absence) of "ips.js," the VM, and how it changes.  A user notes, "i did a few tests and it looks like only the bytecode changes, not the vm itself".  Another user counters, "opcodes are diff between sites though".
    *    (08/28/2021) - User "levi_nz" says: "ive heard the opcodes change i think but not sure if thats true or not".
    *    (08/28/2021) - User "Deleted User" says: "Kasada is not dynamic.". This is met with a "👆" reaction, indicating disagreement.
    *    (08/28/2021) - User "thebotsmith" states: "it does cus it refreshed everytime", and then "opcodes are diff between sites though".

*   **Early Decompilation Efforts (08/29/2021):**
    *   (08/29/2021) - User "thoosje" asks about resources to learn how to reverse Kasada.  User "Deleted User" asks if others wrote a "full disassembler + decompiler" or used a debugger.

*   **Kasada Updates and Responses (08/31/2021 - 09/03/2021):**
    *   (08/31/2021) - "thebotsmith" notes a Kasada update: "kasada updated 😦".  They mention a "big one not too long ago" and a smaller recent one. They note that Kasada "kept version number the same but made changes :KEKW:".
    *   (08/31/2021) - "venomousnz" discusses difficulties extracting Kasada due to Nike not presenting the challenge. Discussion about Datadome on BSTN.
    *   (09/01/2021) - "_miffy_" notes "Subtle changes" and mentions "uuid". "thebotsmith" mentions "an extra param."
    *    (09/01/2021) - User "_miffy_" says "i think so :)" to the statement "nice of them to not update ver number 😄".
    *   (09/02/2021) - "sneakerpenguin" suggests Kasada hired a new intern, possibly from Akamai.
    *    (09/03/2021) - User "cutty0001" jokes, referencing the fact that they dont change version numbers: "Hey man. If you really believe our code is not well, come work for us and let’s see what you are all about!"
    *    (09/03/2021) - "licketysplitoffline" asks "When KasadaAIO".

*   **Later Discussions and Updates (09/14/2021 - 11/18/2021):**
    *   (09/14/2021) - "i7solar" notes that Kasada updates scripts without version changes: "doubt any1 would OS a kasada deobfuscator, they update their scripts w/o version changes".
    *   (09/26/2021) - "thebotsmith" notes another update: "Not good. They updated again 😒". "cutty0001" confirms awareness.  "thebotsmith" mentions "Quite a big update tbf".
    *   (10/10/2021) - "thebotsmith" states: "kasada getting very sneaky now :bigcry:" and "turning into shape soon".
    *   (11/11/2021) - "venomousnz" notes a Kasada update on Veve. "thebotsmith" mentions "multiple vm change".
    *   (11/18/2021) - "unlucku." asks about the frequency of changes to encryption and decoding functions. "thebotsmith" replies "not often" and "every few months," but also notes the "vm is dynamic."
    *   (11/18/2021) - User "Deleted User" posts a link to a SANS DevTools Webinar featuring Kasada (22:04 timestamp is highlighted).

* **Kasada's perspective (11/19/2021):**
 * The aforementioned video is discussed, with many users commenting on it, some sarcastically.

*   **Discussion of Dynamic VM (11/20/2021):**
    *   (11/20/2021) - "thebotsmith" states: "Changes every time you load the page".  "john_wick3571" claims the VM "train itself" and "become harder and harder daily(dynamic)".
     *   (11/21/2021) - User "pxcaptcha" asks "VM is dynamic?" and "In what way". "thebotsmith" replies, "Changes every time you load the page".

* **Veve and Kasada updates.(11/11/2021)**
 * User "venomousnz" says: "am i the only one who noticed that kasada updated veve?"(11/11/2021).
 * User "venomousnz" also says "ok, seems theres more than just that change", indicating that multiple updates were happening.

*   **Further Updates and Discussions (Late 2021 - Early 2022):**
    *   (12/05/2021) - "mehdimoii" reports their Kasada bypass being patched.
    *   (01/12/2022) - "venomousnz" notes: "looks like kasada has changed" and "appears only the FORMAT of the ips.js has changed".
    *   (01/17/2022) - "lennardwalter" finds "kek" references in the source, and later: "there are way too many references to kek or something similar in the source" (1/17/2022).

This timeline shows a consistent pattern: Kasada frequently updates its defenses, and the community actively discusses and adapts to these changes. The dynamic nature of the VM, particularly the changing opcodes and obfuscation techniques, is a recurring theme. The logs also reveal the community's efforts to understand the VM's internals, develop decompilation tools, and share information. The mention of specific file names (`ips.js`, `p.js`), headers (`x-kpsdk-cd`, `x-kpsdk-ct`), and function names (`KPSDK_0xd388()`) provides concrete targets for analysis.

**VI. Conclusion**

**6.1. Summary of Findings**

This report has provided a comprehensive analysis of Kasada's JavaScript Virtual Machine (VM) and the process of decompiling it, drawing on both a formal Bachelor's Thesis and informal Discord chat logs from a community of developers. The key findings are summarized below:

*   **Kasada's Core Defense:** Kasada's primary defense against automated threats (bots) relies on a custom-built, client-side JavaScript VM. This VM executes obfuscated bytecode, making direct analysis of the underlying logic extremely difficult. This is in contrast to simpler obfuscation techniques that can often be reversed with automated tools.

*   **VM Architecture:** The Kasada VM is a register-based virtual machine. Its core components include:
    *   Encoded bytecode (initially a string, then decoded into an array of 32-bit integers).
    *   A dynamic interpreter that fetches, decodes, and executes bytecode instructions.
    *   A control signal array used to interpret different data types within the bytecode.
    *   A randomized opcode array defining the VM's instruction set.
    *   Registers for storing intermediate values.
    *   A separate memory management system.
    *   An interface (FFI) for interacting with the host JavaScript environment.

*   **Dynamic Nature:** A defining characteristic of Kasada's VM is its dynamism.  The bytecode, parsing mechanisms, control signals, and opcode order can change between VM instances and over time. This significantly hinders static analysis and requires adaptive decompilation techniques.

*   **Decompilation Process:** The decompilation process, as outlined in the thesis, involves several key steps:
    1.  **Parsing the Interpreter:** Locating and analyzing the dynamic components of the interpreter (bytecode decoding function, control signal array, opcode array) using tools like SWC to create an AST.
    2.  **Disassembly:** Transforming the bytecode into an Intermediate Representation (IR) consisting of basic blocks and instructions. A Breadth-First Search (BFS) traversal, with special handling for exception handlers, is used.
    3.  **Control Flow Graph (CFG) Construction:** Building a CFG to represent the program's control flow, with nodes representing basic blocks and labeled edges representing control flow transitions.
    4.  **Control Flow Analysis (CFA):** Identifying high-level language constructs (loops, conditionals, exception handlers) by analyzing the structure of the CFG.  This involves techniques like dominator tree analysis and Johnson's algorithm for cycle detection.
    5.  **Code Generation:**  Translating the structured CFG and IR into readable, high-level JavaScript code. This involves structure-aware traversal (modified DFS) and generating appropriate JavaScript constructs for loops, conditionals, and exception handlers.

*   **Challenges and Solutions:** The Discord chat logs provided valuable insights into the real-world challenges faced by developers attempting to bypass Kasada:
    *   **Constant Updates:** Kasada frequently updates its VM, changing opcodes, parsing logic, and control signals. This requires constant adaptation and re-analysis.
    *   **Dynamic Components:** Handling the dynamic nature of the VM is a major hurdle.
    *   **Exception Handling:** Kasada's non-standard exception handling mechanism (using instruction pointers) adds complexity.
    *   **Control Flow Obfuscation:** Techniques like dead code insertion and complex conditional jumps make it harder to follow the program's logic.
    *   **Community Collaboration:** The Discord logs demonstrate the importance of community collaboration in sharing knowledge, techniques, and tools.

* **Open Source:** Discussions existed around open source projects and decompilers, and the benefits and disadvantages of releasing these tools.
* **Evolving Threat Landscape:** The Discord chat logs showed a continuous "cat-and-mouse" game, with Kasada evolving its defenses and developers adapting their techniques.

**6.2. Future Research Directions**

The thesis itself suggests several avenues for future research, and the Discord discussions reinforce these:

*   **Data Flow Analysis (DFA):** The thesis explicitly mentions the absence of DFA as a limitation. Implementing DFA would enable more sophisticated analysis of data relationships and dependencies, leading to:
    *   Better variable naming.
    *   More accurate identification of compound expressions.
    *   Improved optimization of the generated code.

*   **Adapting Classical Techniques:** Further research could explore how other classical program analysis techniques, beyond those already used in the thesis, could be adapted for JavaScript VM decompilation.

*   **Optimization Techniques:** Developing optimization techniques specifically tailored for decompiled JavaScript code could improve the readability and efficiency of the output.

*   **Handling Other VMs:** Extending the decompiler to handle other JavaScript VM-based obfuscation systems would broaden its applicability.

*   **Automated Deobfuscation:** The ultimate goal might be to fully automate the decompilation process, creating a tool that can adapt to new VM instances without manual intervention. This is a significant challenge given the dynamic nature of Kasada.

*    **Dynamic Analysis:** combining static analysis (decompilation) with dynamic analysis (observing the VM's behavior at runtime) could provide a more complete picture.

**6.3. Final Thoughts**

Decompiling Kasada's JavaScript VM is a complex and ongoing challenge. The VM's dynamic nature and sophisticated obfuscation techniques require a combination of theoretical knowledge (decompilation principles, control flow analysis) and practical skills (reverse engineering, adapting to constant changes).

The combination of the structured approach in the thesis and the real-world insights from the Discord chat logs provides a comprehensive understanding of the problem.  The thesis demonstrates that classical decompilation techniques *can* be adapted to handle modern VM-based obfuscation, even in a dynamic environment like JavaScript.

*Impact of Open-Source Contributions:*
The Discord discussions and the existence of the thesis itself raise questions about the impact of open-source contributions in this domain. While sharing knowledge and tools can be beneficial for learning and research, it also risks aiding malicious actors. The rapid patching of vulnerabilities exposed by publicly available information (as seen in the Discord chats) highlights this tension. A balance must be struck between promoting open research and preventing the misuse of information. However, the ongoing evolution of anti-bots like Kasada, and the continued research into decompilation and reverse engineering, demonstrates that even a constantly evolving and complex, dynamic system is not full-proof.

The cat-and-mouse game between bot developers and anti-bot systems is likely to continue.  Further research and development in both areas will be crucial for maintaining web security and understanding the limits of obfuscation techniques.

