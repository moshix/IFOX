# How the IFOX Assembler Builds Relocatable Object Files

A guide for non-mainframe experts: passes, ESD, RLD, and the assembly-to-execution pipeline.

---

## Color Key

| Color | Meaning |
|-------|---------|
| <span style="color:#1565c0;font-weight:bold">Source / Input</span> | What goes in |
| <span style="color:#2e7d32;font-weight:bold">Processing</span> | Transformation steps |
| <span style="color:#e65100;font-weight:bold">Metadata (ESD, RLD)</span> | Control information |
| <span style="color:#6a1b9a;font-weight:bold">Machine Code</span> | Executable output |
| <span style="color:#616161;font-weight:bold">Storage / Files</span> | Workfiles, object deck |

---

## The Big Picture: From Source to Running Program

```mermaid
flowchart LR
    subgraph Assembly[" "]
        A[SOURCE PROGRAM<br>.asm files]
    end
    subgraph Link[" "]
        B[RELOCATABLE OBJECT<br>ESD + TXT + RLD]
    end
    subgraph Load[" "]
        C[LOAD MODULE<br>executable]
    end
    subgraph Run[" "]
        D[RUNNING PROGRAM<br>in memory]
    end
    
    A -->|Assembler IFOX| B
    B -->|Linkage Editor| C
    C -->|Loader / Program Fetch| D
    
    style A fill:#e3f2fd
    style B fill:#fff3e0
    style C fill:#f3e5f5
    style D fill:#e8f5e9
```

**Pipeline:** <span style="color:#1565c0">Source</span> → <span style="color:#e65100">Object (ESD+TXT+RLD)</span> → <span style="color:#6a1b9a">Load Module</span> → <span style="color:#2e7d32">Running Program</span>

---

## Part 1: The Assembler's Multi-Pass Design

### Why Multiple Passes?

In the 1970s, mainframe memory was tiny (often 64K–256K bytes). You could not hold a full program and all its tables in memory at once. The solution: **process the source in several passes**, each doing one job and writing intermediate results to disk.

<div style="border:1px solid #616161;padding:12px;margin:12px 0;background:#fafafa;font-family:monospace;">
<strong>1970s CONSTRAINT: Limited Memory</strong>
<ul style="margin:8px 0 0 0;">
<li>System/360 "F" machine = 64K total (20K for OS, ~44K for programs)</li>
<li>Assembler must fit in memory AND process large source files</li>
<li>Solution: Read source once per pass, write to workfiles (SYSUT1-3)</li>
<li>Each phase loads, runs, then is DELETED to free memory for the next</li>
</ul>
</div>

### The IFOX Phase Flow

IFOX uses **phases** (subroutines loaded one at a time) instead of a single monolithic program:

<div style="border:1px solid #2e7d32;padding:16px;margin:12px 0;background:#e8f5e9;">
<div style="text-align:center;font-weight:bold;margin-bottom:12px;">IFOX DRIVER (stays in memory)</div>
<div style="text-align:center;margin-bottom:12px;">LOAD → RUN → DELETE (free memory) → LOAD next...</div>
<div style="display:flex;flex-wrap:wrap;justify-content:center;gap:8px;margin:12px 0;">
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">EDIT (X11)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">DICT RES. (X21)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">GEN (X31)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">SYMBOL RES. (X41/42)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">OUTPUT LISTER (X51)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #1565c0;padding:8px 12px;background:#e3f2fd;">RLD & XREF (X61)</span>
</div>
<div style="border:1px solid #616161;padding:8px;margin-top:12px;background:#f5f5f5;text-align:center;">WORKFILES (SYSUT1, SYSUT2, SYSUT3) — intermediate data between phases</div>
</div>

**Pass A (Edit + Generate):** Read source, expand macros, build symbol tables.  
**Pass B (Dictionary + Generator):** Resolve symbols, generate object code.  
**Output:** Relocatable object deck (80-byte records) to SYSGO (object file).

---

## Part 2: The Relocatable Object File — ESD, TXT, RLD

The assembler writes a **relocatable object module**: machine code plus metadata so the linkage editor and loader can place and fix addresses later.

### Object Module Structure (80-byte card format)

<div style="border:1px solid #616161;padding:8px;margin:12px 0;background:#fafafa;">
<div style="text-align:center;font-weight:bold;">OBJECT MODULE = sequence of 80-byte records (like punched cards)</div>
</div>

<div style="border:2px solid #e65100;padding:12px;margin:12px 0;background:#fff8e1;">
<div style="font-weight:bold;margin-bottom:8px;">ESD (External Symbol Dictionary) — MUST COME FIRST</div>
<div style="border:1px solid #e65100;padding:10px;margin:8px 0;background:#fff;">
Who is defined here? What do we need from elsewhere?<br>
SD=Section, LD=Label/Entry, ER=External Ref, PR=Pseudo-Register
</div>
</div>
<div style="text-align:center;margin:4px 0;">↓</div>

<div style="border:2px solid #6a1b9a;padding:12px;margin:12px 0;background:#f3e5f5;">
<div style="font-weight:bold;margin-bottom:8px;">TXT (Text) — Machine code and data</div>
<div style="border:1px solid #6a1b9a;padding:10px;margin:8px 0;background:#fff;">
Actual bytes to load. Addresses are RELATIVE (not final).
</div>
</div>
<div style="text-align:center;margin:4px 0;">↓</div>

<div style="border:2px solid #e65100;padding:12px;margin:12px 0;background:#fff8e1;">
<div style="font-weight:bold;margin-bottom:8px;">RLD (Relocation & Linkage Dictionary) — Where to fix addresses</div>
<div style="border:1px solid #e65100;padding:10px;margin:8px 0;background:#fff;">
"At offset X in section Y, put the address of symbol Z"
</div>
</div>
<div style="text-align:center;margin:4px 0;">↓</div>

<div style="border:1px solid #616161;padding:8px;margin:12px 0;background:#f5f5f5;text-align:center;">
<strong>END</strong> — End of module
</div>

### ESD: The "Phone Book" of Symbols

<div style="border:2px solid #e65100;padding:12px;margin:12px 0;background:#fff8e1;">
<div style="font-weight:bold;">ESD = External Symbol Dictionary</div>
<div style="margin-bottom:12px;">Answers: "What symbols does this module define? What does it need?"</div>
<table style="width:100%;border-collapse:collapse;">
<tr style="background:#e65100;color:white;"><th style="padding:8px;text-align:left;">TYPE</th><th style="padding:8px;text-align:left;">MEANING</th><th style="padding:8px;text-align:left;">EXAMPLE</th></tr>
<tr style="border-bottom:1px solid #ddd;"><td style="padding:8px;">SD</td><td style="padding:8px;">Section Definition</td><td style="padding:8px;">"MYCODE" is a 100-byte code section</td></tr>
<tr style="border-bottom:1px solid #ddd;"><td style="padding:8px;">LD</td><td style="padding:8px;">Label Definition</td><td style="padding:8px;">"START" is an entry point at offset 0</td></tr>
<tr style="border-bottom:1px solid #ddd;"><td style="padding:8px;">ER</td><td style="padding:8px;">External Reference</td><td style="padding:8px;">"PRINT" is defined somewhere else</td></tr>
<tr><td style="padding:8px;">PR</td><td style="padding:8px;">Pseudo-Register (XD)</td><td style="padding:8px;">"BUF" is a dummy section for addressing</td></tr>
</table>
</div>

### RLD: The "Fix-Up List"

The assembler does **not** know final addresses. It emits **placeholders** and records where they are:

<div style="border:2px solid #e65100;padding:12px;margin:12px 0;background:#fff8e1;">
<div style="font-weight:bold;">RLD = Relocation & Linkage Dictionary</div>
<div style="margin-bottom:12px;">Each RLD entry says: "At this location, put the address of that symbol"</div>
<div style="font-family:monospace;margin:8px 0;">Source: &nbsp;&nbsp;&nbsp;E &nbsp;&nbsp;DC &nbsp;&nbsp;A(EXTERNAL+4) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;← Address constant, value unknown</div>
<div style="text-align:center;margin:4px 0;">↓</div>
<div style="font-family:monospace;margin:8px 0;">RLD entry: R=EXTERNAL's ESDID, P=this section, Address=offset, Length=4, Type=A</div>
<div style="margin-top:12px;padding:8px;background:#fff;border:1px solid #e65100;">
<strong>When the LINKAGE EDITOR runs:</strong>
<ol style="margin:4px 0 0 0;">
<li>It knows where EXTERNAL ended up (from ESD of another module)</li>
<li>It finds the TXT byte at the given offset</li>
<li>It REPLACES the placeholder with the real address</li>
</ol>
</div>
</div>

### How RLD Adjusts Displacements

<div style="display:flex;flex-wrap:wrap;gap:24px;margin:12px 0;">
<div style="flex:1;min-width:200px;border:2px solid #1565c0;padding:12px;background:#e3f2fd;">
<div style="font-weight:bold;margin-bottom:8px;">BEFORE LINK (in object module)</div>
<div style="font-family:monospace;">TXT: [instruction] [????] [instruction] ...</div>
<div style="font-size:0.9em;margin-top:8px;">↑ placeholder (e.g. 0 or relative value)<br>RLD says: "Put address of FOO here"</div>
</div>
<div style="flex:1;min-width:200px;border:2px solid #2e7d32;padding:12px;background:#e8f5e9;">
<div style="font-weight:bold;margin-bottom:8px;">AFTER LINK (linkage editor applies RLD)</div>
<div style="font-family:monospace;">TXT: [instruction] [0x00012345] [instruction] ...</div>
<div style="font-size:0.9em;margin-top:8px;">↑ real address where FOO was placed</div>
</div>
</div>

---

## Part 3: The Assembler, Linkage Editor, and Loader — Working Together

The assembler, linkage editor, and loader form a **pipeline**. Each step adds information the next step needs.

### The Three-Stage Pipeline

```mermaid
flowchart TB
    subgraph S1["STAGE 1: ASSEMBLER"]
        A1[Reads source]
        A2[Builds ESD]
        A3[Generates TXT with placeholders]
        A4[Builds RLD]
        A5[Output: ONE object module per source file]
    end
    
    subgraph S2["STAGE 2: LINKAGE EDITOR"]
        B1[Reads one or more object modules]
        B2[Merges ESDs: matches ER with SD/LD]
        B3[Assigns final placement]
        B4[Applies RLD: fills placeholders]
        B5[Output: ONE load module]
    end
    
    subgraph S3["STAGE 3: LOADER"]
        C1[Reads load module]
        C2[Allocates memory]
        C3[Relocates if needed]
        C4[Transfers control to entry point]
    end
    
    S1 --> S2 --> S3
    
    style S1 fill:#e3f2fd
    style S2 fill:#fff3e0
    style S3 fill:#e8f5e9
```

### How They Use Each Other's Output

<div style="display:flex;flex-wrap:wrap;justify-content:space-between;gap:16px;margin:12px 0;align-items:flex-start;">
<div style="border:2px solid #1565c0;padding:12px;background:#e3f2fd;flex:1;min-width:140px;">
<div style="font-weight:bold;text-align:center;">ASSEMBLER</div>
<div style="font-size:0.9em;margin-top:8px;">
ESD: "I define FOO, I need BAR"<br>
TXT: code + placeholders<br>
RLD: "fix offset 12 with address of BAR"
</div>
</div>
<div style="border:2px solid #e65100;padding:12px;background:#fff8e1;flex:1;min-width:140px;">
<div style="font-weight:bold;text-align:center;">LINKAGE EDITOR</div>
<div style="font-size:0.9em;margin-top:8px;">
Uses ESD to: Match BAR, assign addresses, resolve refs<br>
Produces: Single executable, all refs resolved
</div>
</div>
<div style="border:2px solid #2e7d32;padding:12px;background:#e8f5e9;flex:1;min-width:140px;">
<div style="font-weight:bold;text-align:center;">LOADER</div>
<div style="font-size:0.9em;margin-top:8px;">
Uses load module to: Copy code, relocate, start PC
</div>
</div>
</div>
<div style="display:flex;align-items:center;justify-content:center;gap:12px;margin:12px 0;flex-wrap:wrap;">
<span style="border:1px solid #616161;padding:8px 12px;background:#f5f5f5;">Object Module(s)</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #6a1b9a;padding:8px 12px;background:#f3e5f5;">Load Module</span>
<span style="color:#616161;">→</span>
<span style="border:1px solid #2e7d32;padding:8px 12px;background:#e8f5e9;">Program in Memory</span>
</div>

### Example: Two Modules Linked Together

<div style="display:flex;gap:16px;flex-wrap:wrap;margin:12px 0;">
<div style="flex:1;min-width:200px;border:2px solid #1565c0;padding:12px;background:#e3f2fd;">
<div style="font-weight:bold;">MODULE A (main)</div>
<div style="font-size:0.9em;margin-top:8px;">
ESD: MAIN (SD), CALL SUB (ER)<br>
TXT: BALR R14,R15 ; call SUB<br>
RLD: "R15 = address of SUB"
</div>
</div>
<div style="flex:1;min-width:200px;border:2px solid #1565c0;padding:12px;background:#e3f2fd;">
<div style="font-weight:bold;">MODULE B (subroutine)</div>
<div style="font-size:0.9em;margin-top:8px;">
ESD: SUB (SD, LD)<br>
TXT: ... code for SUB<br>
RLD: (internal only)
</div>
</div>
</div>
<div style="text-align:center;margin:8px 0;">↓</div>
<div style="border:2px solid #e65100;padding:12px;margin:12px 0;background:#fff8e1;">
<div style="font-weight:bold;">LINKAGE EDITOR</div>
<ol style="margin:8px 0 0 0;">
<li>Sees A needs SUB (ER)</li>
<li>Sees B defines SUB (SD/LD)</li>
<li>Places B's code, computes SUB's address</li>
<li>Uses RLD from A to put SUB's address into the call</li>
</ol>
</div>
<div style="text-align:center;margin:8px 0;">↓</div>
<div style="border:2px solid #2e7d32;padding:12px;margin:12px 0;background:#e8f5e9;text-align:center;">
<strong>LOAD MODULE:</strong> One contiguous program with MAIN and SUB, all refs fixed
</div>

### Why This Design Was Smart in the 1970s

| Constraint | How the design responds |
|------------|-------------------------|
| **Little memory** | Assembler uses phases: load one, run, delete, load next. Workfiles hold intermediate data. |
| **Slow CPU** | Each tool does one job. No need to re-parse source during link. |
| **Expensive disk** | 80-byte card format is compact. ESD/RLD are small compared to full symbol tables. |
| **Separate compile** | You can change one module and relink without reassembling everything. |
| **Shared libraries** | Linkage editor can pull in pre-built object libraries (e.g. I/O routines). |

---

## Summary

<div style="display:flex;align-items:center;justify-content:center;gap:12px;flex-wrap:wrap;margin:16px 0;padding:16px;background:#fafafa;border:1px solid #ddd;">
<span style="border:1px solid #1565c0;padding:10px 16px;background:#e3f2fd;"><strong>SOURCE</strong><br><small>.asm files</small></span>
<span style="color:#616161;font-size:1.2em;">→</span>
<span style="border:1px solid #e65100;padding:10px 16px;background:#fff8e1;"><strong>ASSEMBLER</strong><br><small>ESD + TXT + RLD</small></span>
<span style="color:#616161;font-size:1.2em;">→</span>
<span style="border:1px solid #e65100;padding:10px 16px;background:#fff8e1;"><strong>LINKAGE EDITOR</strong><br><small>Load Module</small></span>
<span style="color:#616161;font-size:1.2em;">→</span>
<span style="border:1px solid #2e7d32;padding:10px 16px;background:#e8f5e9;"><strong>LOADER</strong><br><small>Running Program</small></span>
</div>
<div style="display:flex;justify-content:center;gap:8px;margin:8px 0;font-size:0.9em;color:#616161;">
<span>"What & where"</span>
<span>|</span>
<span>"Combine & fix"</span>
<span>|</span>
<span>"Load & go"</span>
</div>

The assembler produces **relocatable** output (ESD + TXT + RLD). The linkage editor **resolves** it into a load module. The loader **places** it in memory and **starts** it. Each stage uses the previous one's output and adds the next layer of binding.
