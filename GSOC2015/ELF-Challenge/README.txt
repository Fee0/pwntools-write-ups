# GOSC 2015 ELF Modification Writeup

# The challenge


The challenge for the ELF modification project was to re-order the PLT and GOT of the "bof" file from pwnable.kr.


# The intent

The intention of re-ordering the PLT/GOT is make it harder to calculate the base address of e.g. libc with a leaked pointer from PLT/GOT. If we know, that in this binary always the n-th entry in the GOT points to the function puts() in libc, than we could calculate the base address of libc and execute ROP-chains and so on. But if we don't know which entry will point to which function we can just guess to which function the n-th entry will point. The bigger the tables the harder it will be to guess the base address of the lib.

# Function calls via PLT/GOT

The "bof" binary only contains three calls into the PLT. The calls can be found via objdump:

objdump --file-offsets --disassemble bof | grep call.*plt
48e:	e8 7d 00 00 00       	call   510 <__gmon_start__@plt> (File Offset: 0x510)
55c:	e8 bf ff ff ff       	call   520 <__libc_start_main@plt> (File Offset: 0x520)
59f:	e8 3c ff ff ff       	call   4e0 <__cxa_finalize@plt> (File Offset: 0x4e0)

Here we see the function and the offset to the corresponding PLT entry. E.g. the PLT entry for __gmon_start__ is 0x7d relative to the address of the byte following this call (0x493) or at offset 0x510 from the beginning of the file.

To actually see the code at this offsets in the PLT we can use the following command:

$ objdump -M intel -d bof

...
Disassembly of section .plt:

000004b0 <gets@plt-0x10>:
 4b0:	ff b3 04 00 00 00    	push   DWORD PTR [ebx+0x4]
 4b6:	ff a3 08 00 00 00    	jmp    DWORD PTR [ebx+0x8]
 4bc:	00 00                	add    BYTE PTR [eax],al
	...

000004c0 <gets@plt>:
 4c0:	ff a3 0c 00 00 00    	jmp    DWORD PTR [ebx+0xc]
 4c6:	68 00 00 00 00       	push   0x0
 4cb:	e9 e0 ff ff ff       	jmp    4b0 <_init+0x3c>

000004d0 <__stack_chk_fail@plt>:
 4d0:	ff a3 10 00 00 00    	jmp    DWORD PTR [ebx+0x10]
 4d6:	68 08 00 00 00       	push   0x8
 4db:	e9 d0 ff ff ff       	jmp    4b0 <_init+0x3c>

000004e0 <__cxa_finalize@plt>:
 4e0:	ff a3 14 00 00 00    	jmp    DWORD PTR [ebx+0x14]
 4e6:	68 10 00 00 00       	push   0x10
 4eb:	e9 c0 ff ff ff       	jmp    4b0 <_init+0x3c>

000004f0 <puts@plt>:
 4f0:	ff a3 18 00 00 00    	jmp    DWORD PTR [ebx+0x18]
 4f6:	68 18 00 00 00       	push   0x18
 4fb:	e9 b0 ff ff ff       	jmp    4b0 <_init+0x3c>

00000500 <system@plt>:
 500:	ff a3 1c 00 00 00    	jmp    DWORD PTR [ebx+0x1c]
 506:	68 20 00 00 00       	push   0x20
 50b:	e9 a0 ff ff ff       	jmp    4b0 <_init+0x3c>

00000510 <__gmon_start__@plt>:
 510:	ff a3 20 00 00 00    	jmp    DWORD PTR [ebx+0x20]
 516:	68 28 00 00 00       	push   0x28
 51b:	e9 90 ff ff ff       	jmp    4b0 <_init+0x3c>

00000520 <__libc_start_main@plt>:
 520:	ff a3 24 00 00 00    	jmp    DWORD PTR [ebx+0x24]
 526:	68 30 00 00 00       	push   0x30
 52b:	e9 80 ff ff ff       	jmp    4b0 <_init+0x3c>



Here we see the PLT. This table actually consists of code. The code in the first entry PLT[0] is responsible for finding the right address of a function for the n-th PLT/GOT entry at runtime. Every other entry in the PLT has the same format and represents a function. 

The first instruction in a PLT entry is a jump to the corresponding GOT entry. The ebx register contains the address to the base of the GOT. The offset added to ebx is used because the first three entries in the GOT are used for other stuff. E.g. the first real entry in the PLT <gets@plt> is associated with the third entry in the GOT since the GOT only contains 4 byte addresses. If this function is called the first time, there is not yet the right address in the GOT. The GOT only contains a pointer back to the instruction followed by the jump (push).
The second instruction is simply a value that is pushed to the stack and serves as argument for the next jump.
The last jump goes always to PLT[0], which then find the right address which is associated with the n-th entry (indicated by the argument on the stack) in the PLT and writes this address to the GOT. The next time this function is called, the first jump in the PLT will directly jump to the address in the GOT, which is this time the right address in the lib.


To actually see the GOT we can type the following:

$ readelf -x .got.plt bof 

Hex-Ausgabe der Sektion ‚.got.plt‛:					
  0x00001ff4 141f0000 00000000 00000000 c6040000 ................	
  0x00002004 d6040000 e6040000 f6040000 06050000 ................	
  0x00002014 16050000 26050000                   ....&...


The important question is, how PLT[0] actually resolves the right address for the function.
This is done with the help of some symbols:

$ readelf -r bof 

Relozierungssektion '.rel.plt' an Offset 0x43c enhält 7 Einträge:
 Offset     Info    Typ             Sym.-Wert  Sym.-Name
00002000  00000107 R_386_JUMP_SLOT   00000000   gets
00002004  00000207 R_386_JUMP_SLOT   00000000   __stack_chk_fail
00002008  00000307 R_386_JUMP_SLOT   00000000   __cxa_finalize		
0000200c  00000407 R_386_JUMP_SLOT   00000000   puts
00002010  00000507 R_386_JUMP_SLOT   00000000   system
00002014  00000607 R_386_JUMP_SLOT   00000000   __gmon_start__		
00002018  00000707 R_386_JUMP_SLOT   00000000   __libc_start_main	

All these offsets points into the GOT. With this information the dynamic loader knows which locations in the GOT needs to be patched for which function.


# How the re-ordering can be done

To reorder the PLT/GOT two things needs to be fixed up.


1.) All calls needs to be fixed, since every call goes directly into the GOT.
2.) Since the dynamic loader (over-) writes the content of the GOT, when it is called the first time, we need to fool him somehow. This is done by rewriting the symbols in the .rel.plt section.


But first, we need make the new order sure. At the beginning, we saw three calls the and the corresponding PLT entries:

__gmon_start__ 	-> PLT[6]
__libc_start_main 	-> PLT[7]
__cxa_finalize 		-> PLT[3]


Now we want to re-order these entries,

PLT[6] -> PLT[7] 
PLT[7] -> PLT[3] 
PLT[3] -> PLT[6] 

respectively these functions

__gmon_start__  		-> 	__libc_start_main
__libc_start_main 		->	__cxa_finalize
__cxa_finalize 			-> 	__gmon_start__

This means that now these functions are associated with the following PLT entries:

__gmon_start__ 	-> PLT[7]
__libc_start_main 	-> PLT[3]
__cxa_finalize 		-> PLT[6]


Now we look again the three calls:

$ objdump --file-offsets --disassemble bof | grep call.*plt
 48e:	e8 7d 00 00 00       	call   510 <__gmon_start__@plt> (File Offset: 0x510)			
 55c:	e8 bf ff ff ff       	call   520 <__libc_start_main@plt> (File Offset: 0x520) 		
 59f:	e8 3c ff ff ff       	call   4e0 <__cxa_finalize@plt> (File Offset: 0x4e0)			

Now we want the first call to __gmon_start__@plt to go to the offset where __libc_start_main@plt goes. Therefore we need to patch the offset after the OPcode (0xe8). 
We can calculate the offset by taking the desired file offset (0x520) - the address of the byte after the call instruction (0x48e + 5 Byte = 0x493). So, 0x520 - 0x493 = 0x8d. So for the first call we need to patch 0x7d000000 with 0x8d000000 (remember endianess). The other call can be fixed accordingly.

The result should look like this:

$ objdump --file-offsets --disassemble bof | grep call.*plt
 48e:	e8 8d 00 00 00       	call   520 <__gmon_start__@plt> (File Offset: 0x520)
 55c:	e8 7f ff ff ff       	call   4e0 <__libc_start_main@plt> (File Offset: 0x4e0)
 59f:	e8 6c ff ff ff       	call   510 <__cxa_finalize@plt> (File Offset: 0x510)


Now the first step is done and every call points to the right (new) PLT entry. Now we need to fool the dynamic loader to put the right address at the right (re-ordered) location in the GOT.

Therefore we look again at the relocation information:

$ readelf -r bof 

Relozierungssektion '.rel.plt' an Offset 0x43c enhält 7 Einträge:
 Offset     Info    Typ             Sym.-Wert  Sym.-Name
00002000  00000107 R_386_JUMP_SLOT   00000000   gets
00002004  00000207 R_386_JUMP_SLOT   00000000   __stack_chk_fail
00002008  00000307 R_386_JUMP_SLOT   00000000   __cxa_finalize		
0000200c  00000407 R_386_JUMP_SLOT   00000000   puts
00002010  00000507 R_386_JUMP_SLOT   00000000   system
00002014  00000607 R_386_JUMP_SLOT   00000000   __gmon_start__		
00002018  00000707 R_386_JUMP_SLOT   00000000   __libc_start_main	

$ readelf -x .rel.plt bof

Hex-Ausgabe der Sektion ‚.rel.plt‛:
  0x0000043c 00200000 07010000 04200000 07020000 . ....... ......
  0x0000044c 08200000 07030000 0c200000 07040000 . ....... ......
  0x0000045c 10200000 07050000 14200000 07060000 . ....... ......
  0x0000046c 18200000 07070000                   . ......


Each entry in this table consists of two four byte values. The first is the offset (in the GOT) where the right symbol needs to be put. The second is some information about the symbols. 

For each of the three functions we need to swap these 8 bytes with the corresponding bytes in this table. E.g. __gmon_start__ should point to where __libc_start_main points now. Therefore we replace the bytes of __gmon_start__ (0x14200000 0x07060000) with the bytes of __libc_start_main (0x08200000 0x07030000). The other entries can be fixed accordingly. 

The result should look this:

$ readelf -r bof_modifycall2

Relozierungssektion '.rel.plt' an Offset 0x43c enhält 7 Einträge:
 Offset     Info    Typ             Sym.-Wert  Sym.-Name
00002000  00000107 R_386_JUMP_SLOT   00000000   gets
00002004  00000207 R_386_JUMP_SLOT   00000000   __stack_chk_fail
00002018  00000707 R_386_JUMP_SLOT   00000000   __libc_start_main
0000200c  00000407 R_386_JUMP_SLOT   00000000   puts
00002010  00000507 R_386_JUMP_SLOT   00000000   system
00002008  00000307 R_386_JUMP_SLOT   00000000   __cxa_finalize
00002014  00000607 R_386_JUMP_SLOT   00000000   __gmon_start__

And now we have re-ordered PLT and GOT. :-)

See the file “obf” for the original file.
See the file “obf_re-ordered” for the re-ordered ELF. 


