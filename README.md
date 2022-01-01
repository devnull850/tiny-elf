# Tiny ELF

Continuing off of my [previous experiment](https://github.com/devnull850/tiny-bin.git), I wanted to continue trying to get the smallest possible ELF binary that will output a message to the terminal. Ideally it would output "Hello, World!" but I did stumble upon something that will do. Since "ELF" is part of an ELF's magic number, I will cheat and use that for the printed text.

With the "string buffer" out of the way it was time for the 3 required components for a valid ELF binary. I first needed an ELF header and program header. I experimented a little and discovered I could overlap the first 8 bytes of the program header with the last 8 bytes of the ELF header to decrease the size of the binary to 112 bytes instead of 120 (64 for the ELF header and 56 for the program header). This will be the size that the binary will be. The actual code will be weaved into the header's space.

The code required to output the string will need to be snaked around the areas of the two headers that the OS will still allow the binary to run without crashing if it contained random value. I discovered the program header's `p_paddr` field, segment's physical address, could have a random value without much complaint from the OS. This was selected as the entry point of the program.

With an entry point selected, it was time to begin weaving the code. I realized that my setup had all the registers zeroed out. This is not a specification from my understanding and thus has the potential to not work on other setups. Since it did for me, I used that to not have to initialize registers to zero, to save the `xor regsiter,register` instruction which is 2 bytes. I decided to add 3 since `%edx` was already zero to again save 2 bytes from the 5 byte `movl $3,%edx` instruction. With 5 bytes consumed in the 8 byte space reserved for the program header's segment's physical address, I used a 2 bytes jump to get another area in the two headers.

```
  50:	ff c7                	inc    %edi
  52:	83 c2 03             	add    $0x3,%edx
  55:	eb d1                	jmp    0x28
```
The `lea` instruction was the largest and most difficult instruction to fit in. I was using 8 byte spaces and its 7 bytes prevented the two additional bytes needed contiguously to jump to another area. After a bit of trial and error, I realized I could write into the e_flags portion of the ELF header from the reserved space for the start of the section header table without crashing. I used this to load the address for the ELF part of the file's magic number into the `$rsi` register. I then performed a jump to another part of the file.

```
  28:	48 8d 35 d2 ff ff ff 	lea    -0x2e(%rip),%rsi
  2e:	eb 37                	jmp    0x68
```

This was a very convenient 8 bytes for the 4 instructions. This worked perfectlywith the 8 byte space reserved for the program header's alignment field.

```
  68:	89 f8                	mov    %edi,%eax
  6a:	0f 05                	syscall 
  6c:	31 ff                	xor    %edi,%edi
  6e:	eb 98                	jmp    0x8
```

The final jump is to the `exit` syscall, to return control back to the OS appropriately.

```
   8:	b8 3c 00 00 00       	mov    $0x3c,%eax
   d:	0f 05                	syscall 
```
