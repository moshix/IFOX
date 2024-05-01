FX (aka IFOX) Assembler used in OS/VS, DOS, VM
==============================================

This is the assembler source code for the.... assembler.  

Assembler XF, also called IFOX assembler, is a mostly compatible upgrade of Assembler F that includes the new System/370 architecture instructions. This version provides a common assembler for OS/VS, DOS/VS and VM systems. Other changes include relaxing restrictions on expressions and macro processing. Assembler XF requires a minimum partition/region size of 64 KB. Yes, that's asking a lot, but in the meantime there are now mainframes with up to 1MB of memory, so 64KB for an assembler is not a totally crazy amount, in my opinion. 

The history of mainframe assemblers
===================================
An assembler makes a “pass” over the source program when the program to be assembled isread completely. A fundamental difference in the designs of ASMF and ASMH is the number of passes each takes to complete an assembly.
* A “phase” is a subset of a pass, where different sets of instructions are loaded to process portions of the data created by previous phases. For example, an initial phase of Pass 1 might read the source program, identify the fields of each statement, compress them, and then write the results to auxiliary
storage.
* An “interlude” is a pass over some internal tables, and usually takes place between or after source “passes.” An example of interlude processing is the sorting and printing of cross references. (We also talk about a “postlude,” which is an interlude after the last pass.)
* By “binding” we mean the substitution of a value for a symbol. Substitution of X'D2' for the MVC mnemonic is one example of binding. The decision that a macro dictionary will be a particular size is also a binding process. Early binding, in general, improves efficiency and decreases flexibility.

Traditionally, the assembly process is described in two passes:
* Pass 1: Determine the amount of object-code storage needed by each statement, adjust a “Location Counter” accordingly, and assign location values to symbols; note all external symbols.
* Pass 2: Use the information collected in Pass 1 to generate object code.  

The IFOX assembler is based on the earlier ASM E and F assemblers. These assemblers were constrained by the small size of early System/360 processors, and many features (and limitations) of today's Assembler Language can be traced to the original design of ASME and ASMF. The memory sizes of System/360 machines was specified with letters: E meant a 32K-byte machine (14K bytes for the system, 16K bytes for applications); F meant 64K bytes (20K for the system), G 128K, and H 256K (56K for the system, 200 for applications like the assembler).

The first (“Pass A”) is called “Edit”: this converts the source program into an internal form more convenient for processing. A special kind of editing, “macro editing” converts macros into an internal form and also constructs macro dictionary descriptors that define the assembly-time storage layout for variable symbols. The fixed-size local dictionary descriptors for macros and for open code, and a global dictionary for global variable symbols, are constructed during this “editing” process.

In addition to the dictionaries, the assembler must build a “symbol table” of ordinary symbols in which to record symbol attributes. In ASMF, two symbol tables are constructed: one during conditional assembly for macro processing (to collect symbol attributes of macro arguments, as needed for macro generation), and one for ordinary assembly (for all symbols). (The non-open source ASMH uses a single symbol table for both purposes);

The term “Generate” is used to describe the (“Pass B”) macro and conditional assembly activity that substitutes macro-generated statements for the occurrence of a macro name in the source program.


The XF assembler specifically 
=============================
The primary benefits of Assembler XF are improved performance compared to ASMF, and some language enhancements made at the same time. The assembler manuals were completely and elegantly rewritten! The reson for the improved performance lies the advanced read-ahead feature of this assembler. 

![Architecture of the XF Assembler](https://github.com/moshix/IFOX/blob/main/xfassembler.png)

Read this manual for a simple guide on how to use this new and user-friendly assembler here: [http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/GC33-4021-1_OS_VS_Assembler_Programmers_Guide_May73.pdf](http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/GC33-4021-4_OS_VS_VM_370_Assembler_Programmers_Guide_Sep82.pdf)

for the whole manual collection go here: http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/

Moshix  
May 1, 2024
