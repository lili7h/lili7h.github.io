---
layout: post
hidden: false
published: true
title: "ADFCSC: Ransomware Writeup"
lead: Don't give in to ransomware!
type: CTF Writeup
image: assets/jpg/martin-sanchez-unsplash-csc.png
alt: Image by Martin Sanchez on Unsplash + the ADFCSC logo overlaid
caption: Image by Martin Sanchez on Unsplash + the ADFCSC logo overlaid
---

Another part of my series of CTF write-ups from ADFCSC 2023, this time a ransomware challenge in the 'Investigation' category.

{% comment %}
LILITH{G17HUB_S0URC3_V13W3R}
{% endcomment %}

| Difficulty Rating | Satisfaction Rating |
| :---------------: | :-----------------: |
| Moderate Low      | Very high!          |
{: .formal_table}

## Set up

> Help! Your friend has had their (100MB) USB drive encrypted with some sort of ransomware attack! The attacker is asking for 💯 bitcoins to decrypt it!! Can you help? Your friend managed to get the encryption binary too.

We have been provided with two zip files:
- encryptorjLBXpK3gz.zip (4 KB)
- volumeC11m7Xo1Zb.zip (100 MB)

These two files inflate to the following:

- encryptor
```bash
$ file encryptor
encryptor: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=9ca7c655b313fe2d63bff9dc2805ad7f3b4d25ae, for GNU/Linux 3.2.0, stripped
```
- volume.bin
```bash
$ file volume.bin
volume.bin: data
```

We can safely assume that the `volume.bin` is a container file for the entire file-system that has been encrypted. Therefor if this was not encrypted and instead recognised by `file` as some form of boot sector or file-system, we would be able to mount it to a folder using the `mount` command and a `loop` device. Something like:
<br />
```bash
$ mkdir test_dir
$ sudo mount -o loop volume.bin test_dir/
```
<br />
For now, we are going to put this file aside and focus on the `encryptor` tool. As it is generally ill-advised to run tools that may encrypt your drives, let's do a bit of reverse engineering and binary analysis first to determine its danger rating. As the old proverb goes, 'everything is a ~~nail~~binary analysis when you have a ~~hammer~~`strings`'. 
<br />
```bash
$ strings encryptor

... snip ...

********************************************************************************
*
                *
*                      THE TARGETED USB STICK DATA LOCKER                      *
* This piece of software is a targeted USB stick data encryptor. It has been   *
* designed to only encrypt a single targeted FAT volume. It will not lock any  *
* FAT volume. Once locked you can then request a ransom.                       *
Enter the path of the FAT volume to encrypt:

... snip ...

```
<br />
This alone is enough to tell us that it isn't going to auto-detect any drives on the system and go around encrypting them. Especially since it _says_ it only targets FAT volumes, of which I have none. Of course, I am still running in a disposable Kali VM, but I trust it enough to spin it up and see what we get.
<br />
```bash
┌──(Lilith㉿kali)-[~/adfctf/Ransom]
└─$ ./encryptor   

********************************************************************************
*                                                                              *
*                      THE TARGETED USB STICK DATA LOCKER                      *
*                                                                              *
* This piece of software is a targeted USB stick data encryptor. It has been   *
* designed to only encrypt a single targeted FAT volume. It will not lock any  *
* FAT volume. Once locked you can then request a ransom.                       *
*                                                                              *
********************************************************************************

Enter the path of the FAT volume to encrypt: volume.bin

[+] Successfully opened specified file for reading.
[+] The size of file to be encrypted is 104857600 bytes.
[+] Volume serial number successfully read.
[!] Requested file is not the expected volume target. Aborting.
```
<br />
Notice how I specified the `volume.bin` file as the target volume to encrypt? It seems that the `encryptor` is expecting files over mounted volumes, so we don't need to run around mounting the device first then specifying the particular `/dev/loopN` device it may be mounted against. This is good, it makes our life just a tiny bit easier. Also note how it does some form of validation against the volume and errors out - since it's reading the volume serial number, we can assume that its using something like the magic byte sequence in the file-system superblock to validate the target. Let's crack open the binary and see what we can work out.

<br />

You might have noticed when we ran `file` on the `extractor` earlier that it is a stripped binary, this only serves to slightly lengthen our reverse engineering process but ultimately will not hinder us much. We will use [Ghidra](https://ghidra-sre.org/) to perform the disassembly and decompilation of the binary. First up, load up a new project and import the binary, then open it in the code viewer and analyse it when prompted, this lands us here:
<br />
![The code viewer landing page after we have analysed the binary.]({% link assets/posts/adfctf/ransom/ghidra-entry.png %})
{% include subtle-caption.html %}
<br />
Since this is a stripped binary (and likely optimised), the assembly isn't going to be that human readable, so lets identify the 'main' function of the program. All ELF (**E**xecutable **L**inking **F**ormat) binaries have an 'entrypoint' function, which is where the program begins _before_ calling the `main` function. Ghidra is always able to identify this 'entrypoint' function regardless of if the binary is stripped due to the fact its address in codespace is hardcoded into the ELF file header. We can use this `entry` function to identify the `main` function, as in a standard C program, the `main` function will be passed as an argument function pointer to the `__libc_start_main` function.
<br />
![The code viewer decompiling the entry function.]({% link assets/posts/adfctf/ransom/ghidra-libc-start-main.png %})
{% include subtle-caption.html %}
<br />
Here we can see a function that Ghidra has identified as `FUN_001014f9` is passed as an argument to `__libc_start_main`, so we can assume this is the main function. Ghidra will let us rename this function, so while we're here, lets rename it to `main`.
<br />
![Renaming the main function. LILITH{L337_H7ML_S0URC3_V13W3R}]({% link assets/posts/adfctf/ransom/ghidra-rename-main.png %})
{% include subtle-caption.html %}
<br />
Now we are able to decompile the main function, which turns out to be about 120 lines of stripped C code, with the important bits being the following:
<br />
```c
  printf("Enter the path of the FAT volume to encrypt: ");
  __isoc99_scanf(&DAT_00102246,local_188);
  putchar(10);
  local_240 = fopen(local_188,"rb");
  if (local_240 == (FILE *)0x0) {
    puts("[!] Unable to open specified file for reading. Aborting.");
    uVar2 = 0xffffffff;
  }
  else {
    puts("[+] Successfully opened specified file for reading.");
    iVar1 = stat(local_188,&local_228);
    if (iVar1 == 0) {
      printf("[+] The size of file to be encrypted is %ld bytes.\n",local_228.st_size);
      local_244 = 0x347726c9;
      fseek(local_240,0x27,0);
      sVar3 = fread(&local_248,4,1,local_240);
      if (sVar3 == 1) {
        puts("[+] Volume serial number successfully read.");
        if (local_244 == local_248) {
          puts("[+] Requested file is the expected volume target.");
          rewind(local_240);
          local_238 = malloc(local_228.st_size);
          sVar3 = fread(local_238,1,local_228.st_size,local_240);
          if (sVar3 == local_228.st_size) {
            puts("[+] Volume successfully read.");
            fclose(local_240);
            local_198 = 0x74b44da6d2c0fe2c;
            local_190 = 0x71528916c1391e5;
            FUN_00101321(local_118,0x10,&local_198);
            local_230 = malloc(local_228.st_size);
            FUN_001013eb(local_118,local_228.st_size,local_238,local_230);
            puts("[+] Volume data successfully encrypted in memory.");
            free(local_238);
            local_240 = fopen(local_188,"wb");
            if (local_240 == (FILE *)0x0) {
              puts("[!] Unable to open specified file for writing. Aborting.");
              free(local_230);
              uVar2 = 0xffffffff;
            }
            else {
              puts("[+] Successfully opened specified file for writing.");
              sVar3 = fwrite(local_230,1,local_228.st_size,local_240);
              if (sVar3 == local_228.st_size) {
                puts("[+] Encrypted volume successfully written.");
                free(local_230);
                fclose(local_240);
                puts("[+] Done.");
                uVar2 = 0;
              }
```
<br />
Now, as you probably very quickly decided, this is a nightmare to read and understand. Variables aren't named, the code isn't succint, there's no comments... And unfortunately the only way to really get around this is to spend the time analysing what the code is doing and renaming variables and functions until it's more readable. I will save you the effort and do it 'off camera'. Here are some important bits I extracted from the above method (bonus points if you can work out which sections correspond to which):
<br />
```c
if (read_status == stat_buffer.st_size) {
  puts("[+] Volume successfully read.");
  fclose(f_status);

  key_password_1 = 0x74b44da6d2c0fe2c;
  key_password_2 = 0x71528916c1391e5;
  key_generator(key, 0x10, &key_password_1); // <----- function of interest 1

  encrypted_volume_buff = malloc(stat_buffer.st_size);
  volume_encrypt( // <----- function of interest 2
    key,
    stat_buffer.st_size,
    plaintext_volume_buff,
    encrypted_volume_buff
  ); 

  puts("[+] Volume data successfully encrypted in memory.");
  free(plaintext_volume_buff);

  // ...
}
``` 
<br />
Now this begins to make a bit more sense. If you notice the first 'function of interest', we have some sort of password based key derivation function (PBKDF), with a hardcoded password. Then we have some form of encryption algorithm that takes this key.
<br />
```c
void volume_encrypt(long key,ulong size,long plaintext,long ciphertext) {
  int byte_ctr = 0;
  int keyed_byte_ctr = 0;

  for (int i = 0; (ulong)(long)i < size; i = i + 1) {
    byte_ctr = (byte_ctr + 1) % 256;
    keyed_byte_ctr = (int)(keyed_byte_ctr + (uint)*(byte *)(key + byte_ctr)) % 256;
    deref_and_swap(key + byte_ctr,keyed_byte_ctr + key);
    *(byte *)(ciphertext + i) =
         *(byte *)(plaintext + i) ^ // <--- operation of interest
         *(byte *)(key + (ulong)(byte)(*(char *)(key + keyed_byte_ctr) + *(char *)(key + byte_ctr) ))
    ;
  }
  return;
}
```
<br />
Ignoring the awful type-casting introduced by the compilation-decompilation process, we can begin to see some form of 'encryption' algorithm forming. Primarily, we are XOR'ing each byte of the plaintext with some transformation of the current byte of the keystream. How exactly we modify this is now irrelevant at this point, since due to the associativity of XOR, the encrypt operation is the same as the decrypt operation. Given we can construct the same key, running the encrypted volume through this function will decrypt it, by associativity.
<br />
```c
void key_generator(long key_buff,int key_len,long password) {
  int key_byte = 0;

  for (int i = 0; i < 256; i = i + 1) {
    *(char *)(key_buff + i) = (char)i;
  }
  for (int k = 0; k < 256; k = k + 1) {
    key_byte = (int)((uint)*(byte *)(password + k % key_len) +
                    (uint)*(byte *)(key_buff + k) + key_byte) % 256;
    deref_and_swap(key_buff + k, key_byte + key_buff);
  }
  return;
}
```
<br />
We can see that, however we are actually constructing the key, there are no random elements involved. Thus the same key will be generated from this PBKDF function every time the `encryptor` is run. Since we are generating the same key, and the encryption function is associative, if we can just run the program on the encrypted volume again, it should decrypt. You might have noticed that upon the first trial run, we errored out when trying this with:
<br />
```c
Enter the path of the FAT volume to encrypt: volume.bin

[+] Successfully opened specified file for reading.
[+] The size of file to be encrypted is 104857600 bytes.
[+] Volume serial number successfully read.
[!] Requested file is not the expected volume target. Aborting.
```
<br />
We can see that this error occurs here:
<br />
```c
read_status = fread(&volume_identifier,4,1,f_status);
if (read_status == 1) {
  puts("[+] Volume serial number successfully read.");
  if (magic_bytes == volume_identifier) {
    // ... do rest of process
  } else {
    puts("[!] Requested file is not the expected volume target. Aborting.");
    fclose(f_status);
    exit_code = -1;
  }
}
```
<br />
Seeing as that `volume_identifier` variable isn't used again after that point and isn't relevant to the function of the program, we can perform a binary modification to invert the condition of the `if` statement to continue the desired control flow. We can safely assume that this check is failing because the encrypted volume obviously no longer has the same magic bytes.
<br />
![Comparing the disassembly and decompilation in Ghidra]({% link assets/posts/adfctf/ransom/ghidra-byte-code.png %})
{% include subtle-caption.html %}
<br />
We can see the `if` condition is encoded as a `cmp` op-code followed by a `jz` opcode. But note that if we invert the condition of the `if` statement, that is the same as inverting the jump targets for the `true` and `false` branches. And _that_ is the same as flipping the jump condition (indicated by the character(s) after the 'j' of the opcode. `jz` indicates 'jump if zero'). 

<br />

| Instruction | Description       | Flags | short jump opcodes | near jump opcodes |
| :---------: | :---------------: | :---: | :----------------: | :---------------: |
| JE          | Jump if equal     | ZF=1  | 74                 | 0F 84             |
| JZ          | Jump if zero      | ZF=1  | 74                 | 0F 84             |
| JNE         | Jump if not equal | ZF=0  | 75                 | 0F 85             |
| JNZ         | Jump if not zero  | ZF=0  | 75                 | 0F 85             |
{: .formal_table}

<br />

This Intel x86 Quick reference table from [Steve Friedl](http://unixwiz.net/techtips/x86-jumps.html) shows that if we want to invert the jump condition, its as simple as changing the `74 28` of the jump opcode to a `75 28`. This is a single byte change that we can implement using the `hexedit` tool in the terminal. 

A quick trick for finding the correct byte in the binary to edit is to search for a hex-string that includes an instruction or two after the targeted instruction, as that is less likely to appear more than once.
<br />
![Viewing the byte-code and disassembly of the targeted if condition.]({% link assets/posts/adfctf/ransom/ghidra-byte-code-search.png %})
{% include subtle-caption.html %}
<br />
Here I am inclined to search for the hex string `74 28 48 8d 05 44 0c 00 00`.
<br />
```bash
$ hexedit encryptor
```
<br />
![Searching for a hexstring in hexedit]({% link assets/posts/adfctf/ransom/hexedit-search-hexstring.png %})
{% include subtle-caption.html %}
<br />
![Making the modification at the target]({% link assets/posts/adfctf/ransom/hexedit-write-new.png %})
{% include subtle-caption.html %}
<br />
Notice the 'bolded' `75` indicating a change has been made.

Now that we've made that change, we can run the program against the encrypted volume.
<br />
```bash
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ ./encryptor 

********************************************************************************
*                                                                              *
*                      THE TARGETED USB STICK DATA LOCKER                      *
*                                                                              *
* This piece of software is a targeted USB stick data encryptor. It has been   *
* designed to only encrypt a single targeted FAT volume. It will not lock any  *
* FAT volume. Once locked you can then request a ransom.                       *
*                                                                              *
********************************************************************************

Enter the path of the FAT volume to encrypt: volume.bin

[+] Successfully opened specified file for reading.
[+] The size of file to be encrypted is 104857600 bytes.
[+] Volume serial number successfully read.
[+] Requested file is the expected volume target.
[+] Volume successfully read.
[+] Volume data successfully encrypted in memory.
[+] Successfully opened specified file for writing.
[+] Encrypted volume successfully written.
[+] Done.
```
<br />
Great! We ran right through to completion and didn't encounter anymore errors. Lets check the volume now:
```bash
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ file volume.bin    
volume.bin: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "mkfs.fat", sectors/cluster 4, reserved sectors 4, root entries 512, Media descriptor 0xf8, sectors/FAT 200, sectors/track 32, heads 8, sectors 204800 (volumes > 32 MB), serial number 0x347726c9, unlabeled, FAT (16 bit)
```
<br />
Great! We now recognise the volume as a disk boot sector, and later in the `file` output it identifies as a FAT volume. Lets try and mount it against a loop device.
```bash
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ mkdir testdir   
                                                                       
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ sudo mount -o loop volume.bin testdir                                                
[sudo] password for lilith: 
```
<br />
`mount` didn't return any errors, so lets try and read it!
<br />
```bash
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ ls testdir 
flag.txt
      
┌──(lilith㉿kali)-[~/adfctf/Ransom]
└─$ cat testdir/flag.txt 
FLAG{YoUCanTExT0rTMe!}
```
<br />
Amazing!

So to recap, we:
- Identified the tool doesn't behave maliciously on its own
- Reverse engineered the functions in the stripped `encryptor` binary
- Identified a hard-coded password, and a PBKDF being used to generate the encryption keys
- Identified that the encryption function was associative
- Performed a binary modification on the `encryptor` to allow us to run it on unrecognised volumes
- Used the modified binary to decrypt the encrypted volume by associativity
- Extracted the flag from the newly decrypted volume.