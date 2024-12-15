[![Hits](https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fgithub.com%2Fmoshix%2FIFOX&count_bg=%2379C83D&title_bg=%23555555&icon=ibm.svg&icon_color=%23E7E7E7&title=hits&edge_flat=false)](https://hits.seeyoufarm.com)

The IBM FX, or IFOX, assembler used in MVS 3.8, DOS/VS, VM/370
==============================================================

This is the assembler source code for the.... assembler.  

Assembler XF, also called IFOX assembler, is a mostly compatible upgrade of Assembler F, also called ASMF, that includes the new [System/370 architecture instructions](https://www.ece.ucdavis.edu/~vojin/CLASSES/EEC272/S2005/Papers/Padegs-IBM360-sep81.pdf), including the instructions pertaining to [virtual memory and virtual machines](https://www.vm.ibm.com/history/50th/s370eavm.pdf). The [IBM S/370 Principles of Operation](http://www.bitsavers.org/pdf/ibm/370/princOps/GA22-7000-0_370_Principles_Of_Operation_Jun70.pdf) is the authoritative compendium describing the architecture covered by the FX assembler. This version provides a common assembler for OS/VS, DOS/VS and VM systems. Other improvements include relaxing restrictions on expressions and macro processing.
  
Assembler XF requires a **minimum region size of 64 KB**. Yes, that's asking a lot, but in the meantime there are now mainframes with up to 16TB of RAM, so 64KB for an assembler is not a dingdong crazy amount, in my opinion, especially considering that the XF assembler still runs nicely on the latest z/OS versions, on z16 hardware. 

Assembler XF was developed at the IBM lab in Lidingö, Sweden in the mid 1970s. It was the assembler supplied to customers with OS/VS2 MVS 3.8, as well as with VM/370 and DOS/VS. It was a complete re-write of the Assembler F which came with the MVT operating system. The primary benefits of Assembler XF are improved performance compared to ASMF, with some language enhancements made at the same time. The assembler manuals were completely and elegantly rewritten. The reson for the greatly improved performance lies in the advanced read-ahead feature of this assembler.

The history of IBM mainframe assemblers
=======================================
The IFOX assembler is based on the earlier ASM E and F assemblers, in syntax, but it is a complete rewrite. These assemblers were constrained by the small size of early System/360 processors, and many features (and limitations) of today's assembler language can be traced to the original design of ASME and ASMF. The memory sizes of System/360 machines was specified with letters:  
- E for a 32K-byte machine (14K bytes for the system, 16K bytes for applications)
- F for 64K bytes (20K for the system) 
- G for 128K
- H for 256K (56K for the system, 200K for applications)
   
[This](https://github.com/moshix/mvs/blob/master/WTOMOSHE.jcl) is a typical example of a program written for the FX assembler. 

MVS/SP, VM/SP and VSE used a later assembler, called ASMH for H sized machines. [Here](https://github.com/moshix/mvs/blob/9d625695c727f610f84cf7ccb3ebc28e3153633f/QUEENS_ASMH#L4) is a program written for AMSH. ASMH is vastly superior to the FX assembler. It allows symbols of up to 63 characters (versus 8 in FX assembler). AMSH allows both unary and binary operators in expressions, and it accepts modifier expressions in literal, e.g., =CL(L’SYM)’#’. In ASMH, literals can be used in EQU statements such as "A EQU =A(ABC)" and in expressions such as "IC 0,=X’010203’-1". And a lot more. 

Quite a few after-market assemblers for the IBM mainframe were written by thid parties. Just a small sample of these would the the [ASSIST assembler](https://faculty.cs.niu.edu/~byrnes/csci360/ho/asusergd.shtml) (also ASSIST for MS-DOS [here](https://github.com/moshix/mvs/blob/master/assist.exe) ), the [Waterloo ASMG](https://github.com/moshix/ASMG) assembler, the [SLAC assembler mods](https://www.gsf-soft.com/Documents/SLAC-MODS.html)  and many, many more, including my own assemblers written in REXX, C, Python, GO, and in bash shell script.   

From the ASMH assembler, IBM then begat the High Level Assembler, [HLASM](https://www.ibm.com/products/high-level-assembler-and-toolkit-feature). Here is a [nice video](https://youtu.be/ESjOPKt9fHw?si=FhpggmWq7C7mwI6r) introducing the HLASM assembler by one of its creators. And that's where we still are today. I doubt IBM will release new mainframe assemblers in the future, though, as the mainframe is slowly, but surely going extinct. And this is why this repository exists. 
  
Making a pass
=============
An assembler makes a “pass” over the source program when the program to be assembled is read completely. A fundamental difference in the designs of ASMF and ASMH is the number of passes each takes to complete an assembly.
* A “phase” is a subset of a pass, where different sets of instructions are loaded to process portions of the data created by previous phases. For example, an initial phase of Pass 1 might read the source program, identify the fields of each statement, compress them, and then write the results to auxiliary
storage.
* An “interlude” is a pass over some internal tables, and usually takes place between or after source “passes.” An example of interlude processing is the sorting and printing of cross references. (We also talk about a “postlude,” which is an interlude after the last pass.)
* By “binding” we mean the substitution of a value for a symbol. Substitution of X'D2' for the MVC mnemonic is one example of binding. The decision that a macro dictionary will be a particular size is also a binding process. Early binding, in general, improves efficiency and decreases flexibility.

Traditionally, the assembly process is described in two passes:
* Pass 1: Determine the amount of object-code storage needed by each statement, adjust a “Location Counter” accordingly, and assign location values to symbols; note all external symbols.
* Pass 2: Use the information collected in Pass 1 to generate object code.  


The first (“Pass A”) is called “Edit”: this converts the source program into an internal form more convenient for processing. A special kind of editing, “macro editing” converts macros into an internal form and also constructs macro dictionary descriptors that define the assembly-time storage layout for variable symbols. The fixed-size local dictionary descriptors for macros and for open code, and a global dictionary for global variable symbols, are constructed during this “editing” process.

In addition to the dictionaries, the assembler must build a “symbol table” of ordinary symbols in which to record symbol attributes. In ASMF, two symbol tables are constructed: one during conditional assembly for macro processing (to collect symbol attributes of macro arguments, as needed for macro generation), and one for ordinary assembly (for all symbols). (The non-open source ASMH uses a single symbol table for both purposes);

The term “Generate” is used to describe the (“Pass B”) macro and conditional assembly activity that substitutes macro-generated statements for the occurrence of a macro name in the source program.

![Architecture of the XF Assembler](https://github.com/moshix/IFOX/blob/main/xfassembler.png)

Read this manual for a simple guide on how to use this new and user-friendly assembler here: [http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/GC33-4021-1_OS_VS_Assembler_Programmers_Guide_May73.pdf](http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/GC33-4021-4_OS_VS_VM_370_Assembler_Programmers_Guide_Sep82.pdf)

For the whole manual collection go [here](http://www.bitsavers.org/pdf/ibm/370/OS_VS/assembler/).

About the IFOX source code
==========================
Here is a description of the function of the various source modules:  
- IFOX0A : This module is the driver for the assembler. this module treats all other phases as subroutines.
- IFOX0B : this module is the workfile i/o package for the assembly. the
other phases interface with this module for all workfile i/o re-
quests and core management
- IFOX0C : this module is the master common work area for the assembler. the module is mapped according to the jcommon dsect. the module is loaded by the driver and remains in core until the end of job. register 13 always points to to this module.
- IFOX0D : this module initializes the assembler. it formats the time, date and assembler level, checks for valid parameters, checks to see that all necessary dd cards are present, and sets the maximum workfile record size.
- IFOX0E : this module is the input common work area for the input i/o pack- age. the is load by the driver. register r7 points to this module while the input i/o package has control.
- IFOX0F  : this module is the input i/o module for the assembler. it is used by the macro editor to read source input, copy code and macros.
- IFOX0G  : this module is the output common work area for the output i/o package. this is loaded by the driver. register r7 points to this module while the input i/o package has control.
- IFOX0H  : this module is the output package for the assembler. the other phases interface with this module by use of the jprint and jpunch macros.
- IFOX0I  : this module is called when an irrecoverable error exists. it is entered for permanent i/o errors, missing dd cards (sysprint, sysut1, sysut2, sysut3, sysin), insufficient memory and certain program logic errors. this routine frees all core, closes all files, freepool as necessary, writes a message to the operator and deletes all loaded phases except itself and common.
- IFOX0J  : A table of parameter options for assembler xf.
this table is referenced by the assembler xf initialization phase (ifox02).
the parameter options are those which may be specified by the
programmer in the 'parm' field of the 'exec' statement. this
table also provides for a standard set of defaule parameters when none
is specified.a standard version of this table is furnished with the assembler.
However, the system programmer may reassemble this
table and respecify other default options to meet the requirements of his
installation. In addition, options may be 'fixed' to prevent their misuse.
'fixed' options should be avoided because they limit the programmer's prerogatives.
They should be used only when their inverse function may cause incompatible
installation procedures.

More about mainframe assemblers in this interesting read here: ([https://github.com/moshix/IFOX/blob/main/MTS_Assemblers_Sep1986.pdf](https://github.com/moshix/IFOX/blob/main/MTS_Assemblers_Sep1986.pdf)]

Moshix  
Vermont, December 2024
