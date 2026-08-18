# Lib-Dumper-


╔══════════════════════════════════════════════════════════════╗
║        sourcecodelibbsd.so — ELF Lib Dumper Guide          ║
╚══════════════════════════════════════════════════════════════╝
1. BUILD
bash
# Install GCC if missing
sudo apt install gcc -y        # Debian/Ubuntu
sudo pacman -S gcc             # Arch
sudo yum install gcc -y        # RHEL/CentOS

# Compile
gcc -o libdumper libdumper.c -ldl

# Make executable
chmod +x libdumper
2. BASIC USAGE
bash
# Full dump — pulls everything
./libdumper libg.so

# Dynamic symbols only (stripped binary)
./libdumper libg.so --no-static

# Skip runtime dlopen load
./libdumper libg.so --no-runtime

# Header + sections only, no symbols
./libdumper libg.so --no-static --no-dynamic --no-runtime
3. READING THE OUTPUT
ELF Header
Type:         DYN        ← shared library (.so)
              EXEC       ← executable
              REL        ← relocatable object (.o)

Machine:      0xb7       ← ARM64 (AArch64)
              0x3e       ← x86_64
              0x28       ← ARM32

Entry point:  0x0        ← 0 = shared lib, no entry
              0x401000   ← executable entry point

Endian:       Little     ← x86/ARM standard
              Big        ← MIPS/SPARC/PowerPC

OS/ABI:       Linux      ← built for Linux
              SystemV    ← generic UNIX ABI
Section Table
IDX  NAME              TYPE        OFFSET            SIZE
0    (null)            NULL        0x0               0
1    .text             PROGBITS    0x1000            0x4200    ← executable code
2    .rodata           PROGBITS    0x5200            0x800     ← read-only data, strings
3    .data             PROGBITS    0x6000            0x200     ← initialized globals
4    .bss              NOBITS      0x6200            0x100     ← zero-initialized globals
5    .dynsym           DYNSYM      0x400             0x300     ← exported symbols
6    .dynstr           STRTAB      0x700             0x180     ← symbol name strings
7    .dynamic          DYNAMIC     0x8000            0x1e0     ← linker metadata
8    .plt              PROGBITS    0x900             0x80      ← procedure linkage table
9    .got              PROGBITS    0x980             0x60      ← global offset table

Flags column:

W = Writable      (.data, .bss, .got)
A = Allocated     (loaded into memory at runtime)
X = Executable    (.text, .plt)
M = Mergeable     (.rodata strings)
S = Strings       (null-terminated string data)
T = TLS           (thread-local storage)
Program Headers (Segments)
IDX  TYPE      OFFSET            VADDR              FILESZ      FLAGS
0    LOAD      0x0               0x0                0x1000      R--   ← read-only load
1    LOAD      0x1000            0x1000             0x5200      R-X   ← executable load
2    LOAD      0x6000            0x6000             0x300       RW-   ← read-write load
3    DYNAMIC   0x8000            0x8000             0x1e0       RW-   ← dynamic linker info
4    NOTE      0x200             0x200              0x44        R--   ← build ID, ABI tag

FLAGS:

R = Readable
W = Writable
X = Executable
RWX on same segment = red flag (shellcode injection surface)
Dynamic Section
NEEDED       liblog.so          ← runtime dependency
NEEDED       libc.so.6          ← another dependency
SONAME       libg.so            ← this library's own name
RPATH        /data/local/tmp    ← where linker searches first (hijack surface)
INIT         0x1200             ← runs before main — constructor
FINI         0x1300             ← runs after main — destructor
STRTAB       0x700              ← string table address
SYMTAB       0x400              ← symbol table address

What to look for:

NEEDED lines = every lib this binary depends on
RPATH = hijackable if world-writable path
INIT/FINI = code that runs automatically — priority target for hooks
Dynamic Symbols (.dynsym) — GREEN = function
IDX    VALUE              SIZE      BIND      TYPE      VIS       NAME
1      0x0000000000000000 0         GLOBAL    FUNC      DEFAULT   Java_com_game_NativeLib_init
2      0x0000000000001240 284       GLOBAL    FUNC      DEFAULT   processInput
3      0x0000000000001360 96        GLOBAL    FUNC      DEFAULT   getPlayerState
4      0x0000000000001400 512       GLOBAL    FUNC      DEFAULT   updatePosition
5      0x0000000000000000 0         WEAK      FUNC      DEFAULT   __cxa_finalize

Column breakdown:

VALUE  = virtual address of the symbol (0x0 = imported, not defined here)
SIZE   = byte size of the function/object
BIND   = GLOBAL (exported), LOCAL (internal), WEAK (overridable)
TYPE   = FUNC (function), OBJECT (variable/struct), NOTYPE (unknown)
VIS    = DEFAULT (visible), HIDDEN (not exported), PROTECTED (visible, not preemptable)
NAME   = symbol name — JNI functions start with Java_

JNI pattern:

Java_com_brawlstars_NativeLib_getHealth
      ↑               ↑           ↑
   package         class name   method name
Static Symbols (.symtab)
# Only present in unstripped binaries
# Stripped binary → "No .symtab section" message

IDX    VALUE              SIZE      BIND      TYPE      NAME
45     0x0000000000001800 128       LOCAL     FUNC      calculate_damage
46     0x0000000000001880 64        LOCAL     FUNC      validate_token
47     0x0000000000001900 256       GLOBAL    OBJECT    g_player_data

Static symbols reveal internal functions not exported in .dynsym — the ones the developer didn't intend to expose.

Runtime Dump
[+] Loaded at: 0x7f4a2b000000    ← actual base address in memory
[+] Path:      /data/app/com.supercell.brawlstars/lib/arm64/libg.so

Base address + symbol VALUE = real runtime address of any function.

# Example:
Base:          0x7f4a2b000000
processInput:  +0x1240
Real address:  0x7f4a2b001240   ← hook this with Frida/MemoryPatcher
4. WORKFLOW — FULL ANALYSIS
bash
# Step 1: Full dump, save output
./libdumper libg.so > libg_dump.txt 2>&1

# Step 2: Find all exported functions
grep "FUNC" libg_dump.txt | grep "GLOBAL"

# Step 3: Find JNI entry points
grep "Java_" libg_dump.txt

# Step 4: Find dependencies
grep "NEEDED" libg_dump.txt

# Step 5: Check for suspicious RPATH
grep "RPATH" libg_dump.txt

# Step 6: Look for interesting objects (game state structs)
grep "OBJECT" libg_dump.txt
5. WHAT THE SYMBOLS TELL YOU
Pattern	Meaning
Java_*	JNI bridge — called directly from Java/Kotlin
_Z*	C++ mangled name — run c++filt to demangle
__init_array	Runs at load time — before any Java code
SIZE = 0	Imported symbol — defined in another lib
VIS = HIDDEN	Internal — not in export table but in memory
BIND = WEAK	Overridable at link time
RPATH set	Linker search path — potential hijack surface

Demangle C++ names:

bash
echo "_ZN6Player11updateStateEv" | c++filt
# Output: Player::updateState()
6. COMBINE WITH OTHER TOOLS
bash
# Cross-reference with objdump disassembly
objdump -d libg.so | grep -A 20 "processInput"

# String dump — find hardcoded values
strings libg.so | grep -E "(token|key|secret|url|http)"

# Hex dump a specific section
dd if=libg.so bs=1 skip=$((0x1240)) count=284 | xxd

# readelf cross-check
readelf -s libg.so     # symbols
readelf -d libg.so     # dynamic section
readelf -S libg.so     # section headers
