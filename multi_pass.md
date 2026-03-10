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
        A[SOURCE PROGRAM<br>cards or dataset]
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

<small>Note: On S/370, source files have no special extension — the system uses DD names (SYSIN, SYSLIB) and dataset names, not file extensions like .asm.</small>

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

<p style="margin-bottom:16px;">The object module is a <strong>sequence of 80-byte records</strong> — like a deck of punched cards, read top to bottom. Each record type has a specific role:</p>

<div style="max-width:560px;margin:0 auto 24px auto;position:relative;">
  <!-- Card 1: ESD -->
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:1.1em;margin-bottom:6px;">ESD</div>
    <div style="font-size:0.85em;color:#5d4037;">External Symbol Dictionary — <em>must come first</em></div>
    <div style="margin-top:10px;padding:10px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.9em;">
      Who is defined here? What do we need from elsewhere?<br>
      <span style="font-family:monospace;font-size:0.85em;">SD</span> Section &nbsp; <span style="font-family:monospace;font-size:0.85em;">LD</span> Label &nbsp; <span style="font-family:monospace;font-size:0.85em;">ER</span> External Ref &nbsp; <span style="font-family:monospace;font-size:0.85em;">PR</span> Pseudo-Register
    </div>
  </div>
  <!-- Connector -->
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <!-- Card 2: TXT -->
  <div style="border-left:4px solid #6a1b9a;background:linear-gradient(135deg,#f3e5f5 0%,#e1bee7 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#4a148c;font-size:1.1em;margin-bottom:6px;">TXT</div>
    <div style="font-size:0.85em;color:#5d4037;">Machine code and data</div>
    <div style="margin-top:10px;padding:10px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.9em;">
      Actual bytes to load. Addresses are <strong>relative</strong> (not final).
    </div>
  </div>
  <!-- Connector -->
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <!-- Card 3: RLD -->
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:1.1em;margin-bottom:6px;">RLD</div>
    <div style="font-size:0.85em;color:#5d4037;">Relocation &amp; Linkage Dictionary</div>
    <div style="margin-top:10px;padding:10px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.9em;">
      "At offset X in section Y, put the address of symbol Z"
    </div>
  </div>
  <!-- Connector -->
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <!-- Card 4: END -->
  <div style="border-left:4px solid #616161;background:linear-gradient(135deg,#f5f5f5 0%,#eeeeee 100%);padding:12px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#424242;font-size:1em;">END</div>
    <div style="font-size:0.85em;color:#616161;">End of module</div>
  </div>
</div>

<div style="text-align:center;margin-top:8px;font-size:0.85em;color:#757575;">
  <svg width="120" height="24" viewBox="0 0 120 24" style="vertical-align:middle;margin-right:4px;">
    <rect x="0" y="4" width="80" height="16" fill="#e0e0e0" stroke="#9e9e9e" stroke-width="1" rx="2"/>
    <line x1="10" y1="8" x2="70" y2="8" stroke="#9e9e9e" stroke-width="0.5"/>
    <line x1="10" y1="12" x2="70" y2="12" stroke="#9e9e9e" stroke-width="0.5"/>
    <line x1="10" y1="16" x2="70" y2="16" stroke="#9e9e9e" stroke-width="0.5"/>
    <text x="40" y="22" font-size="6" fill="#616161" text-anchor="middle">80 bytes</text>
  </svg>
  Each record = one 80-byte card
</div>

### ESD: The "Phone Book" of Symbols

<div style="max-width:560px;margin:0 auto 24px auto;">
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:1.1em;margin-bottom:6px;">ESD = External Symbol Dictionary</div>
    <div style="font-size:0.9em;color:#5d4037;margin-bottom:12px;">Answers: "What symbols does this module define? What does it need?"</div>
    <table style="width:100%;border-collapse:collapse;background:rgba(255,255,255,0.7);border-radius:4px;overflow:hidden;">
      <tr style="background:#e65100;color:white;"><th style="padding:10px;text-align:left;">TYPE</th><th style="padding:10px;text-align:left;">MEANING</th><th style="padding:10px;text-align:left;">EXAMPLE</th></tr>
      <tr style="border-bottom:1px solid #ddd;"><td style="padding:10px;">SD</td><td style="padding:10px;">Section Definition</td><td style="padding:10px;">"MYCODE" is a 100-byte code section</td></tr>
      <tr style="border-bottom:1px solid #ddd;"><td style="padding:10px;">LD</td><td style="padding:10px;">Label Definition</td><td style="padding:10px;">"START" is an entry point at offset 0</td></tr>
      <tr style="border-bottom:1px solid #ddd;"><td style="padding:10px;">ER</td><td style="padding:10px;">External Reference</td><td style="padding:10px;">"PRINT" is defined somewhere else</td></tr>
      <tr><td style="padding:10px;">PR</td><td style="padding:10px;">Pseudo-Register (XD)</td><td style="padding:10px;">"BUF" is a dummy section for addressing</td></tr>
    </table>
  </div>
</div>

### RLD: The "Fix-Up List"

The assembler does **not** know final addresses. It emits **placeholders** and records where they are:

<div style="max-width:560px;margin:0 auto 24px auto;position:relative;">
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:1.1em;margin-bottom:6px;">RLD = Relocation &amp; Linkage Dictionary</div>
    <div style="font-size:0.9em;color:#5d4037;margin-bottom:12px;">Each RLD entry says: "At this location, put the address of that symbol"</div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #6a1b9a;background:linear-gradient(135deg,#f3e5f5 0%,#e1bee7 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#4a148c;font-size:0.95em;margin-bottom:8px;">Source</div>
    <div style="font-family:monospace;padding:10px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.9em;">
      E &nbsp;&nbsp;DC &nbsp;&nbsp;A(EXTERNAL+4)
    </div>
    <div style="font-size:0.85em;color:#5d4037;margin-top:8px;">Address constant — value unknown at assembly time</div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:0.95em;margin-bottom:8px;">RLD entry</div>
    <div style="font-family:monospace;padding:10px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.85em;">
      R=EXTERNAL's ESDID, P=this section, Address=offset, Length=4, Type=A
    </div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #2e7d32;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#1b5e20;font-size:0.95em;margin-bottom:8px;">When the LINKAGE EDITOR runs</div>
    <ol style="margin:8px 0 0 0;padding-left:20px;">
      <li style="margin-bottom:4px;">It knows where EXTERNAL ended up (from ESD of another module)</li>
      <li style="margin-bottom:4px;">It finds the TXT byte at the given offset</li>
      <li>It <strong>REPLACES</strong> the placeholder with the real address</li>
    </ol>
  </div>
</div>

### How RLD Adjusts Displacements

<div style="max-width:560px;margin:0 auto 24px auto;position:relative;">
  <div style="border-left:4px solid #1565c0;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#0d47a1;font-size:1em;margin-bottom:8px;">BEFORE LINK</div>
    <div style="font-size:0.85em;color:#5d4037;margin-bottom:8px;">In the object module — placeholder in TXT</div>
    <div style="font-family:monospace;padding:12px;background:rgba(255,255,255,0.8);border-radius:4px;font-size:0.9em;letter-spacing:1px;">
      [instruction] <span style="background:#ffcdd2;padding:2px 6px;border-radius:2px;">????</span> [instruction] ...
    </div>
    <div style="font-size:0.85em;color:#5d4037;margin-top:8px;">RLD says: "Put address of FOO here"</div>
  </div>
  <div style="height:28px;border-left:2px dashed #9e9e9e;margin-left:20px;position:relative;">
    <span style="position:absolute;left:28px;top:4px;font-size:0.8em;color:#757575;">linkage editor applies RLD</span>
  </div>
  <div style="border-left:4px solid #2e7d32;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#1b5e20;font-size:1em;margin-bottom:8px;">AFTER LINK</div>
    <div style="font-size:0.85em;color:#5d4037;margin-bottom:8px;">Placeholder replaced with real address</div>
    <div style="font-family:monospace;padding:12px;background:rgba(255,255,255,0.8);border-radius:4px;font-size:0.9em;letter-spacing:1px;">
      [instruction] <span style="background:#c8e6c9;padding:2px 6px;border-radius:2px;">0x00012345</span> [instruction] ...
    </div>
    <div style="font-size:0.85em;color:#5d4037;margin-top:8px;">Real address where FOO was placed by the linkage editor</div>
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

<div style="max-width:560px;margin:0 auto 32px auto;">
  <div style="border-left:4px solid #1565c0;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#0d47a1;margin-bottom:10px;font-size:1.1em;">1. ASSEMBLER</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Input:</strong> Source deck (cards or dataset)</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Produces:</strong></div>
    <ul style="margin:0 0 8px 0;padding-left:20px;font-size:0.9em;">
      <li><strong>ESD</strong> — defines symbols, lists external refs</li>
      <li><strong>TXT</strong> — machine code with placeholders</li>
      <li><strong>RLD</strong> — where each placeholder goes</li>
    </ul>
    <div style="font-size:0.85em;color:#5d4037;">Output: Object module</div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;margin-bottom:10px;font-size:1.1em;">2. LINKAGE EDITOR</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Input:</strong> Object module(s)</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Uses ESD to:</strong></div>
    <ul style="margin:0 0 8px 0;padding-left:20px;font-size:0.9em;">
      <li>Match external refs to definitions</li>
      <li>Assign final addresses</li>
      <li>Apply RLD — fill placeholders</li>
    </ul>
    <div style="font-size:0.85em;color:#5d4037;">Output: Load module</div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #2e7d32;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);padding:16px;margin-bottom:0;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#1b5e20;margin-bottom:10px;font-size:1.1em;">3. LOADER</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Input:</strong> Load module</div>
    <div style="font-size:0.9em;margin-bottom:8px;"><strong>Does:</strong></div>
    <ul style="margin:0 0 8px 0;padding-left:20px;font-size:0.9em;">
      <li>Allocates memory</li>
      <li>Copies code, relocates if needed</li>
      <li>Starts program at entry point</li>
    </ul>
    <div style="font-size:0.85em;color:#5d4037;">Output: Running program</div>
  </div>
  <div style="display:flex;align-items:center;justify-content:center;gap:8px;flex-wrap:wrap;margin-top:16px;padding:12px;background:#fafafa;border-radius:8px;">
    <span style="font-size:0.9em;color:#616161;">Data flow:</span>
    <span style="border:1px solid #616161;padding:6px 10px;background:#fff;border-radius:4px;font-size:0.9em;">Object Module(s)</span>
    <span style="color:#9e9e9e;">→</span>
    <span style="border:1px solid #6a1b9a;padding:6px 10px;background:#f3e5f5;border-radius:4px;font-size:0.9em;">Load Module</span>
    <span style="color:#9e9e9e;">→</span>
    <span style="border:1px solid #2e7d32;padding:6px 10px;background:#e8f5e9;border-radius:4px;font-size:0.9em;">Program in Memory</span>
  </div>
</div>

### Concrete Example: S/370 Program and Object Module Construction

Here is a minimal S/370 program split into two modules. Source is typically on punched cards or in a dataset — **file extensions have no meaning** on S/370; the system uses DD names (SYSIN, SYSLIB) and member names.

<div style="max-width:640px;margin:0 auto 24px auto;">
  <div style="border-left:4px solid #1565c0;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);padding:16px;margin-bottom:12px;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#0d47a1;margin-bottom:8px;">Source deck 1 — MAIN (calls SUB)</div>
    <pre style="margin:0;font-size:0.85em;background:rgba(255,255,255,0.8);padding:12px;border-radius:4px;overflow-x:auto;">MAIN     CSECT
         EXTRN SUB
         STM   14,12,12(13)
         LR    12,15
         USING MAIN,12
         L     15,=A(SUB)
         BALR  14,15
         LM    14,12,12(13)
         BR    14
         LTORG
         END   MAIN</pre>
  </div>
  <div style="border-left:4px solid #1565c0;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);padding:16px;margin-bottom:12px;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#0d47a1;margin-bottom:8px;">Source deck 2 — SUB (subroutine)</div>
    <pre style="margin:0;font-size:0.85em;background:rgba(255,255,255,0.8);padding:12px;border-radius:4px;overflow-x:auto;">SUB      CSECT
         ENTRY SUB
         BR    14
         END   SUB</pre>
  </div>
</div>

<div style="max-width:640px;margin:0 auto 32px auto;">
  <div style="font-weight:bold;color:#0d47a1;margin-bottom:12px;font-size:1.1em;">Step 1 — Assembler produces object for MAIN</div>
  <div style="display:flex;align-items:center;gap:12px;margin-bottom:16px;flex-wrap:wrap;">
    <div style="border:2px solid #1565c0;padding:12px 16px;background:#e3f2fd;border-radius:8px;">MAIN source</div>
    <div style="font-size:1.2em;color:#9e9e9e;">→</div>
    <div style="border:2px solid #2e7d32;padding:12px 16px;background:#e8f5e9;border-radius:8px;">Assembler</div>
  </div>
  <div style="font-size:0.9em;color:#616161;margin-bottom:8px;">Object module structure:</div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:0.95em;">ESD</div>
    <div style="font-size:0.9em;margin-top:4px;">SD MAIN (section) · ER SUB ("I need SUB")</div>
  </div>
  <div style="height:12px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #6a1b9a;background:linear-gradient(135deg,#f3e5f5 0%,#e1bee7 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#4a148c;font-size:0.95em;">TXT</div>
    <div style="font-size:0.9em;margin-top:4px;">STM, LR, L, BALR, LM, BR</div>
    <div style="margin-top:6px;padding:6px 8px;background:rgba(255,255,255,0.7);border-radius:4px;font-size:0.85em;">L 15,=A(SUB) → <span style="background:#ffcdd2;padding:2px 4px;">placeholder</span> (SUB unknown)</div>
  </div>
  <div style="height:12px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:0.95em;">RLD</div>
    <div style="font-size:0.9em;margin-top:4px;">At offset of =A(SUB) → put SUB's address when known</div>
  </div>
</div>

<div style="max-width:640px;margin:0 auto 32px auto;">
  <div style="font-weight:bold;color:#0d47a1;margin-bottom:12px;font-size:1.1em;">Step 2 — Assembler produces object for SUB</div>
  <div style="display:flex;align-items:center;gap:12px;margin-bottom:16px;flex-wrap:wrap;">
    <div style="border:2px solid #1565c0;padding:12px 16px;background:#e3f2fd;border-radius:8px;">SUB source</div>
    <div style="font-size:1.2em;color:#9e9e9e;">→</div>
    <div style="border:2px solid #2e7d32;padding:12px 16px;background:#e8f5e9;border-radius:8px;">Assembler</div>
  </div>
  <div style="font-size:0.9em;color:#616161;margin-bottom:8px;">Object module structure:</div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#bf360c;font-size:0.95em;">ESD</div>
    <div style="font-size:0.9em;margin-top:4px;">SD SUB (section) · LD SUB (entry point)</div>
  </div>
  <div style="height:12px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #6a1b9a;background:linear-gradient(135deg,#f3e5f5 0%,#e1bee7 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#4a148c;font-size:0.95em;">TXT</div>
    <div style="font-size:0.9em;margin-top:4px;">BR 14 — no placeholders</div>
  </div>
  <div style="height:12px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #9e9e9e;background:linear-gradient(135deg,#f5f5f5 0%,#eeeeee 100%);padding:14px;margin-bottom:0;box-shadow:0 2px 6px rgba(0,0,0,0.06);border-radius:0 6px 6px 0;">
    <div style="font-weight:bold;color:#616161;font-size:0.95em;">RLD</div>
    <div style="font-size:0.9em;margin-top:4px;color:#757575;">(none — no address constants)</div>
  </div>
</div>

<div style="max-width:640px;margin:0 auto 32px auto;">
  <div style="font-weight:bold;color:#bf360c;margin-bottom:12px;font-size:1.1em;">Step 3 — Linkage editor combines and resolves</div>
  <div style="display:flex;align-items:stretch;gap:12px;margin-bottom:16px;flex-wrap:wrap;">
    <div style="flex:1;min-width:120px;border:2px solid #1565c0;padding:12px;background:#e3f2fd;border-radius:8px;text-align:center;">
      <div style="font-size:0.8em;color:#616161;margin-bottom:4px;">Input</div>
      <div style="font-weight:bold;">MAIN object</div>
      <div style="font-size:0.8em;margin-top:4px;">ER SUB</div>
    </div>
    <div style="flex:1;min-width:120px;border:2px solid #1565c0;padding:12px;background:#e3f2fd;border-radius:8px;text-align:center;">
      <div style="font-size:0.8em;color:#616161;margin-bottom:4px;">Input</div>
      <div style="font-weight:bold;">SUB object</div>
      <div style="font-size:0.8em;margin-top:4px;">SD SUB, LD SUB</div>
    </div>
  </div>
  <div style="border-left:4px solid #e65100;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);padding:16px;margin-bottom:12px;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#bf360c;margin-bottom:12px;">LINKAGE EDITOR</div>
    <div style="display:flex;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:12px;">
      <span style="padding:6px 10px;background:#ffecb3;border-radius:4px;font-size:0.9em;">1. Match</span>
      <span style="font-size:0.9em;">ER SUB</span>
      <span style="color:#e65100;font-weight:bold;">↔</span>
      <span style="font-size:0.9em;">SD SUB</span>
    </div>
    <div style="display:flex;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:12px;">
      <span style="padding:6px 10px;background:#ffecb3;border-radius:4px;font-size:0.9em;">2. Place</span>
      <span style="font-size:0.9em;">MAIN + SUB in load module, compute SUB's address</span>
    </div>
    <div style="display:flex;align-items:center;gap:12px;flex-wrap:wrap;">
      <span style="padding:6px 10px;background:#ffecb3;border-radius:4px;font-size:0.9em;">3. Apply RLD</span>
      <span style="font-size:0.9em;">Replace placeholder in MAIN with SUB's real address</span>
    </div>
  </div>
  <div style="height:16px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border-left:4px solid #2e7d32;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);padding:16px;box-shadow:0 2px 8px rgba(0,0,0,0.08);border-radius:0 8px 8px 0;">
    <div style="font-weight:bold;color:#1b5e20;">LOAD MODULE</div>
    <div style="font-size:0.9em;margin-top:6px;">One executable: MAIN + SUB, all refs resolved. Loader copies into memory and starts at MAIN.</div>
  </div>
</div>

### Two Modules — Visual Summary

<div style="max-width:640px;margin:0 auto 32px auto;">
  <div style="display:grid;grid-template-columns:1fr 1fr;gap:16px;margin-bottom:20px;">
    <div style="border:2px solid #1565c0;border-radius:8px;overflow:hidden;box-shadow:0 2px 8px rgba(0,0,0,0.08);">
      <div style="background:#1565c0;color:white;padding:8px 12px;font-weight:bold;text-align:center;">MAIN (object)</div>
      <div style="padding:12px;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);">
        <div style="border-left:3px solid #e65100;padding:8px 10px;margin-bottom:6px;background:#fff8e1;font-size:0.9em;"><strong>ESD</strong> SD MAIN, ER SUB</div>
        <div style="border-left:3px solid #6a1b9a;padding:8px 10px;margin-bottom:6px;background:#f3e5f5;font-size:0.9em;"><strong>TXT</strong> code + <span style="background:#ffcdd2;padding:1px 3px;">placeholder</span></div>
        <div style="border-left:3px solid #e65100;padding:8px 10px;background:#fff8e1;font-size:0.9em;"><strong>RLD</strong> fix =A(SUB)</div>
      </div>
    </div>
    <div style="border:2px solid #1565c0;border-radius:8px;overflow:hidden;box-shadow:0 2px 8px rgba(0,0,0,0.08);">
      <div style="background:#1565c0;color:white;padding:8px 12px;font-weight:bold;text-align:center;">SUB (object)</div>
      <div style="padding:12px;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);">
        <div style="border-left:3px solid #e65100;padding:8px 10px;margin-bottom:6px;background:#fff8e1;font-size:0.9em;"><strong>ESD</strong> SD SUB, LD SUB</div>
        <div style="border-left:3px solid #6a1b9a;padding:8px 10px;margin-bottom:6px;background:#f3e5f5;font-size:0.9em;"><strong>TXT</strong> BR 14</div>
        <div style="border-left:3px solid #9e9e9e;padding:8px 10px;background:#f5f5f5;font-size:0.9em;color:#757575;"><strong>RLD</strong> none</div>
      </div>
    </div>
  </div>
  <div style="text-align:center;margin:12px 0;">
    <div style="display:inline-block;padding:8px 20px;background:#e65100;color:white;border-radius:8px;font-weight:bold;">LINKAGE EDITOR</div>
    <div style="font-size:0.9em;margin-top:8px;color:#5d4037;">ER SUB ↔ SD SUB · places both · applies RLD</div>
  </div>
  <div style="height:20px;border-left:2px dashed #9e9e9e;margin-left:20px;"></div>
  <div style="border:2px solid #2e7d32;border-radius:8px;overflow:hidden;box-shadow:0 2px 8px rgba(0,0,0,0.08);">
    <div style="background:#2e7d32;color:white;padding:10px 12px;font-weight:bold;text-align:center;">LOAD MODULE</div>
    <div style="padding:12px;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);text-align:center;font-size:0.95em;">MAIN + SUB in one executable · all refs fixed</div>
  </div>
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

<div style="max-width:560px;margin:0 auto 24px auto;">
  <div style="display:flex;align-items:center;justify-content:center;flex-wrap:wrap;gap:0;margin:16px 0;">
    <div style="border-left:4px solid #1565c0;padding:12px 16px;background:linear-gradient(135deg,#e3f2fd 0%,#bbdefb 100%);border-radius:0 6px 6px 0;box-shadow:0 2px 6px rgba(0,0,0,0.06);"><strong>SOURCE</strong><br><small>cards / dataset</small></div>
    <div style="width:20px;height:2px;background:#9e9e9e;flex-shrink:0;"></div>
    <div style="border-left:4px solid #e65100;padding:12px 16px;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);border-radius:0 6px 6px 0;box-shadow:0 2px 6px rgba(0,0,0,0.06);"><strong>ASSEMBLER</strong><br><small>ESD + TXT + RLD</small></div>
    <div style="width:20px;height:2px;background:#9e9e9e;flex-shrink:0;"></div>
    <div style="border-left:4px solid #e65100;padding:12px 16px;background:linear-gradient(135deg,#fff8e1 0%,#ffecb3 100%);border-radius:0 6px 6px 0;box-shadow:0 2px 6px rgba(0,0,0,0.06);"><strong>LINKAGE EDITOR</strong><br><small>Load Module</small></div>
    <div style="width:20px;height:2px;background:#9e9e9e;flex-shrink:0;"></div>
    <div style="border-left:4px solid #2e7d32;padding:12px 16px;background:linear-gradient(135deg,#e8f5e9 0%,#c8e6c9 100%);border-radius:0 6px 6px 0;box-shadow:0 2px 6px rgba(0,0,0,0.06);"><strong>LOADER</strong><br><small>Running Program</small></div>
  </div>
  <div style="display:flex;justify-content:center;gap:16px;margin:12px 0;font-size:0.9em;color:#616161;flex-wrap:wrap;">
    <span>"What &amp; where"</span>
    <span>"Combine &amp; fix"</span>
    <span>"Load &amp; go"</span>
  </div>
</div>

The assembler produces **relocatable** output (ESD + TXT + RLD). The linkage editor **resolves** it into a load module. The loader **places** it in memory and **starts** it. Each stage uses the previous one's output and adds the next layer of binding.
