Using an LLM as a High-Level Compiler: The MenuBuilder Experiment

I've spent over 25 years doing hardware and software development in embedded systems, building complex FPGA and DSP solutions on Xilinx platforms, and writing the firmware and device drivers that make it all work. Schematic capture, PCB layout, VHDL, Verilog, embedded C, Linux kernel work, power supply design — the kind of work where you stare at timing diagrams, pull-up resistor values, and triple-check your reset sequencing before you let anything near real hardware.

Years ago I wrote my own menu compiler tool in C. It was a small utility that read a menu-description file — basically a list of commands, parameters, and help text — and spit out generated C code for a simple parser framework. I used it on project after project. It worked fine for the common case, but every time I wanted to change something about the output — formatting, presentation, how help text was aligned, how optional parameters were handled — I had to dig back into the compiler tool itself, modify the code-generation logic, rebuild, debug, and then re-test all of its output. The compiler was doing its job, but it was rigid. Changing the shape of the generated code meant changing the tool, and that was always more work than it should have been.

So when large language models started getting good enough to write real code, I had a specific question: could I rewrite my menu compiler tool not in C, but in markdown? Could I take the transformation rules that were hard-coded in my old tool and express them as a structured specification document — one that an LLM like Codex or Claude could read alongside a menu-description file and produce the same kind of generated C code, but with far more flexibility? Not just a parser framework, but complete test harnesses, callback hooks, build scripts, and documentation — a significant leap from what my old compiler tool could do, without writing a single line of code-generation logic.

The answer, it turns out, is yes. But only if you tell it exactly what to do of course.

That's what MenuBuilder is: a markdown file that acts as a specification — a set of agent instructions — that turns a large language model into a deterministic C code generator. You give it a simple text file describing a command menu. It gives you back a complete, compilable, embedded-style command parser in C. Every time. No hand-holding, no back-and-forth prompt tweaking. This article walks through how it works, why I built it this way, and what it demonstrates about using LLMs as something closer to a compiler than a chatbot.


THE PROBLEM

If you've worked on embedded systems, you've written a command-line menu before. Maybe it's a debug console over UART. Maybe it's a factory test interface. Maybe it's a bench tool for a data acquisition board. The pattern is always the same: define some commands with keywords and parameters, tokenize the input string, match tokens against a command table, and dispatch to a callback function.

Now, there are classical tools that solve the general parsing problem. Yacc and Bison generate parsers from formal grammars — they're powerful, but they were designed for building programming language compilers, not a 10-command debug menu on a microcontroller. Flex handles lexical analysis and pairs with Yacc, but now you're writing tokenizer rules in one language and grammar rules in another. ANTLR is more modern and generates parsers in multiple target languages, but it brings its own runtime library and a learning curve that assumes you're comfortable with formal language theory. GNU Gengetopt is lighter weight but focused on command-line option parsing, not interactive menu systems. Every one of these tools is built for a bigger problem than what most embedded engineers actually have.

The reality is this: when you're working on a small embedded system and all you need is a simple interactive command menu — something where a technician or engineer can type "DAQ START" and have it do the right thing — you don't want to learn a parser generator framework. You don't want to install Java for ANTLR or figure out Bison's shift-reduce conflicts. You want a flat, readable description of your commands, and you want compilable C code out the other end. That's it.

That's exactly the problem I solved years ago with my C-based menu compiler tool. A plain text file in, generated C code out. No grammar files, no external runtimes, no formal language theory required. Anybody on the team could look at the command description file and understand it instantly. The advantage of a simple flat menu format is that it's accessible to everyone — hardware engineers, test technicians, junior firmware developers — not just the one person on the team who happens to know how parser generators work.

But that tool could only generate the parser itself. Adding a test harness, callback stubs, build scripts, or formatted documentation meant more hand-written code in the generator — and more maintenance every time the output format needed to change. MenuBuilder.md solves the same core problem — simple description in, complete C code out — but with the LLM doing the heavy lifting, the output is far richer and the spec is far easier to modify than rewriting C code in a compiler tool.


THE SOURCE: A PLAIN TEXT COMMAND FILE

The input to MenuBuilder is a .txt file. Nothing fancy. Each line defines one command in this format: FUNCTION:COMMAND:DESCRIPTION. Three fields, separated by colons. That's the entire "source language."

Here's a real example — a basic data acquisition command set:

DaqInit:DAQ INIT [RATE_HZ] {CHANNEL_MASK}:Initialize acquisition rate and optional channel mask
DaqStart:DAQ START {DURATION_MS}:Start acquisition, optionally for a fixed duration
DaqStop:DAQ STOP:Stop acquisition
DaqRead:DAQ READ [CHANNEL]:Read latest sample for a channel
DaqSetGain:DAQ SET GAIN [CHANNEL] [GAIN_DB]:Set channel gain in dB
DaqSetTrigger:DAQ SET TRIGGER [SOURCE] {LEVEL}:Set trigger source and optional level
DaqStatus:DAQ STATUS {DETAIL}:Show DAQ status
DaqCalibrate:DAQ CALIBRATE [CHANNEL] {REFERENCE_MV}:Calibrate a channel with optional reference value
DaqSaveConfig:DAQ SAVE CONFIG [NAME]:Save current configuration profile
DaqLoadConfig:DAQ LOAD CONFIG [NAME]:Load configuration profile

Square brackets mean required parameters. Curly braces mean optional. Keywords are literal text that must match. Comments start with #. That's the whole specification from the user's side.


THE AGENT INSTRUCTIONS: MENUBUILDER.MD

Here's where it gets interesting. The LLM doesn't just "figure out" how to write a menu parser. It follows a detailed specification document — MenuBuilder.md — that describes every aspect of the generated code. This markdown file is the real engine. It's roughly 650 lines of precise, unambiguous rules.

It covers command syntax parsing — how [PARAM], {PARAM}, and keyword tokens are classified. It defines file generation rules — exactly which files to produce: menu.h, menu.c, menu_callback_Test.c, main.c, Makefile, build.bat, and a documentation file. It specifies parser correctness requirements, including explicit match signaling with a "matched" flag. The spec actually calls out the specific bug pattern to avoid — treating a return code of 0 as a match indicator — because that's a failure mode I've seen LLMs produce when the instruction isn't clear enough.

The spec also covers dispatch architecture (pre-parsed command metadata, token-count bounds for fast rejection, case-insensitive keyword comparison at exact positions), commenting and licensing (every file gets an LGPL 2.1 header, Doxygen-style function comments, revision history tables), a complete build system (build.bat and Makefile templates with -Wall -Wextra -Werror -pedantic so the generated code has to compile clean), help output formatting (runtime-computed column alignment so help text lines up regardless of command length), and documentation generation (a MenuDocumentation.md file with Mermaid diagrams showing architecture, control flow, data structures, and a complete command reference table).

The point is: the markdown file doesn't leave room for interpretation. It reads more like a compiler specification than a prompt. When the LLM processes a user's .txt file against this spec, it's doing something that looks a lot like compilation — reading a defined input grammar, applying transformation rules, and emitting structured output in a target language.


WHAT COMES OUT

From that 10-line .txt file, MenuBuilder produces a complete project. menu.h provides the public API — structs, typedefs, callback forward declarations, macros for buffer sizes and token limits. menu.c contains the parser and dispatcher — tokenizer, syntax pre-parser, match-and-dispatch loop with explicit match flag, and aligned help printer. menu_callback_Test.c has stub callback implementations that print parsed arguments back to the console. main.c is a desktop test harness with a --demo mode that can run canned commands or accept interactive input. The Makefile handles Linux builds and build.bat handles Windows — same targets, same compiler flags. MenuDocumentation.md is a full design document with Mermaid architecture diagrams, dispatch flowcharts, data structure class diagrams, and a command reference table.

You run build.bat on Windows or make on Linux, and you get a working menu.exe that accepts commands, parses parameters, matches keywords case-insensitively, and dispatches to the right callback. Type HELP or ? and you get a neatly aligned command listing. Type an invalid command and you get a clear error. It handles optional parameters, rejects missing required parameters, and won't false-match on token count alone.

The generated code compiles under -Werror with zero warnings. That's not a nice-to-have — it's a hard requirement in the spec.


WHY THIS MATTERS: LLMS AS COMPILERS

This is a small project, and I'm not pretending it replaces a real parser generator like yacc or ANTLR. But it demonstrates something worth paying attention to: a well-written specification document can turn a general-purpose LLM into a domain-specific code generator.

The key insight is that the LLM isn't being creative here. It's being constrained. The MenuBuilder.md file removes ambiguity the same way a compiler specification removes ambiguity. The input grammar is defined. The output structure is defined. The edge cases are called out. The bugs to avoid are explicitly documented.

This is a different use of the technology than most people are talking about. It's not "ask the AI to write some code and hope it works." It's more like this: define your input format (the .txt file), define your transformation rules (the .md specification), feed both to the LLM, and get deterministic, correct, compilable output. The .md file is reusable. Swap in a different .txt file with different commands and you get a completely different menu system — same architecture, same quality, same structure. That's compilation behavior.


WHAT I LEARNED

A few practical takeaways from building this.

Specificity is everything. The more precise the spec, the more predictable the output. Vague instructions produce vague code. MenuBuilder.md is 650+ lines for a reason.

Call out the bugs explicitly. The spec includes a "prohibited pattern" section that describes a specific dispatch bug — treating rc == 0 as a match. This isn't paranoia. It's a pattern I've seen LLMs produce when the instruction isn't clear enough. Naming the failure mode prevents it.

Require the code to compile clean. Mandating -Werror in the build scripts means you find out immediately if the generated code has issues. It's a built-in smoke test.

Treat the spec like source code. MenuBuilder.md has a version history, an LGPL license header, and a defined structure. It's maintained the same way you'd maintain any other engineering document. Because it is one.


WHERE THIS GOES

I'm using this same pattern — structured markdown specs driving LLM code generation — for other domains. FPGA register map generators. Test script builders. Documentation systems. The approach scales anywhere you have a well-defined input format and a well-defined output structure.

The tools will keep getting better. The models will keep getting faster. But the engineering discipline of writing a clear spec? That doesn't change. If anything, it matters more now — because the spec is the program.


The full source — the .txt command file, the MenuBuilder.md specification, and an example generated project — is available on GitHub for anyone who wants to try it or adapt it:

https://github.com/sspano01/MenuBuilder


TRY IT YOURSELF

To generate the files in the ExampleMenu folder yourself, open a conversation with an LLM (such as Claude, ChatGPT, or GitHub Copilot) and use the following prompt. Attach both MenuBuilder.md and menu_commands_daq.txt to the conversation, then send:

"Using the attached MenuBuilder.md and menu_commands_daq.txt command file, produce a complete menu, test application, windows build.bat, linux makefile, and full documentation for the generated menu. Place all generated files in a folder called ExampleMenu."

That single prompt — with both files attached — is all it takes. The LLM reads the .txt file as input and applies the transformation rules in MenuBuilder.md to produce:

menu.h — Public API, structs, typedefs, and macros
menu.c — Tokenizer, parser, dispatcher, and help printer
menu_callback_Test.c — Stub callbacks that echo parsed arguments
main.c — Desktop test harness with --demo mode
Makefile — Linux build (gcc, -Wall -Wextra -Werror -pedantic)
build.bat — Windows build (same flags)
MenuDocumentation.md — Design document with Mermaid diagrams and command reference

Once generated, build and run:

On Linux: cd ExampleMenu, then make, then ./menu
On Windows: cd ExampleMenu, then build.bat, then menu.exe

Type HELP or ? at the prompt to see the full command listing, or try commands like DAQ INIT 1000 and DAQ STATUS to exercise the parser.
