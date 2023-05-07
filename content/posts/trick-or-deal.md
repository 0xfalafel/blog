---
title: "Writeup: Trick or Deal"
date: 2023-05-05T19:54:04+02:00
draft: true
---

# Introduction

**Trick or Deal** is *Medium* a retired [hackthebox](https://app.hackthebox.com/challenges/trick-or-deal) ***pwn*** challenge.

This binary has a secret function `unlock_storage()`, and a **Use-After-Free** vulnerability.

To exploit this binary, I will use [pwndbg](https://github.com/pwndbg/pwndbg) and [pwntools](https://docs.pwntools.com/en/stable/).  
[Ghidra](https://ghidra-sre.org/) was also used for the analysis. The snipets of **C** code were obtained using the *Ghidra decompiler*.

# Analysis

The binary give us a **menu** with **5 actions**. Each actions will call a function.

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

The *2nd action* `[2] Buy Weapons` call the fuction **`buy()`** and has **no interest** for us.  
It simply output back what we write.

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

0x555555603000	<font color="#F4BF75">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000291</font>	<font color="#A1EFE4">................</font>
0x555555603010	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#A1EFE4">................</font>
[...]

0x555555603290	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#AE81FF">0x0000000000000061</font>	<font color="#AE81FF">........a.......</font>
0x5555556032a0	<font color="#AE81FF">0x67694c206568540a</font>	<font color="#AE81FF">0x0a72656261737468</font>	<font color="#AE81FF">.The Lightsaber.</font>
0x5555556032b0	<font color="#AE81FF">0x6e6f53206568540a</font>	<font color="#AE81FF">0x7765726353206369</font>	<font color="#AE81FF">.The Sonic Screw</font>
0x5555556032c0	<font color="#AE81FF">0x0a0a726576697264</font>	<font color="#AE81FF">0x0a73726573616850</font>	<font color="#AE81FF">driver..Phasers.</font>
0x5555556032d0	<font color="#AE81FF">0x696f4e206568540a</font>	<font color="#AE81FF">0x6b63697243207973</font>	<font color="#AE81FF">.The Noisy Crick</font>
0x5555556032e0	<font color="#AE81FF">0x00000000000a7465</font>	<font color="#AE81FF">0x0000555555400be6</font>	<font color="#AE81FF">et........@UUU..</font>
0x5555556032f0	<font color="#AE81FF">0x0000000000000000</font>	<font color="#A6E22E">0x0000000000020d11</font>	<font color="#A6E22E">................</font>	 &lt;-- Top chunk
</code></pre></div>
{{< /rawhtml >}}


At the **address `0x555555603290`**, we have a **chunk** of size *0x60*.

#### Mixed data and function pointers

The *chunk* is the variable **`storage`** which contain some **data**, and a **pointer** to **`printStorage()`**: `0x555555400be6`.  
We can see that `0x555555603290` is a pointer to `printStorage()` using the **`disassemble`** command.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#F92672"><b>pwndbg&gt; </b></font>disassemble 0x0000555555400be6
Dump of assembler code for function <font color="#F4BF75">printStorage</font>:
   <font color="#66D9EF">0x0000555555400be6</font> &lt;+0&gt;:	<font color="#A6E22E">push</font><font color="#F8F8F2">   </font><font color="#F92672">rbp</font>
   <font color="#66D9EF">0x0000555555400be7</font> &lt;+1&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rbp</font>,<font color="#F92672">rsp</font>
   <font color="#66D9EF">0x0000555555400bea</font> &lt;+4&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rax</font>,<font color="#F92672">QWORD</font><font color="#F8F8F2"> </font><font color="#F92672">PTR</font><font color="#F8F8F2"> </font>[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x20144f</font>]<font color="#F8F8F2">        # 0x555555602040 &lt;storage&gt;</font>
   <font color="#66D9EF">0x0000555555400bf1</font> &lt;+11&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rdx</font>,<font color="#F92672">rax</font>
   <font color="#66D9EF">0x0000555555400bf4</font> &lt;+14&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rax</font>,<font color="#F92672">QWORD</font><font color="#F8F8F2"> </font><font color="#F92672">PTR</font><font color="#F8F8F2"> </font>[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x201425</font>]<font color="#F8F8F2">        # 0x555555602020 &lt;stdout@@GLIBC_2.2.5&gt;</font>
   <font color="#66D9EF">0x0000555555400bfb</font> &lt;+21&gt;:	<font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">r8</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x63f</font>]<font color="#F8F8F2">        # 0x555555401241</font>
   <font color="#66D9EF">0x0000555555400c02</font> &lt;+28&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rcx</font>,<font color="#F92672">rdx</font>
   <font color="#66D9EF">0x0000555555400c05</font> &lt;+31&gt;:	<font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">rdx</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x67f</font>]<font color="#F8F8F2">        # 0x55555540128b</font>
   <font color="#66D9EF">0x0000555555400c0c</font> &lt;+38&gt;:	<font color="#A6E22E">lea</font><font color="#F8F8F2">    </font><font color="#F92672">rsi</font>,[<font color="#F92672">rip</font><font color="#F92672"><u style="text-decoration-style:single">+</u></font><font color="#66D9EF">0x6ad</font>]<font color="#F8F8F2">        # 0x5555554012c0</font>
   <font color="#66D9EF">0x0000555555400c13</font> &lt;+45&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">rdi</font>,<font color="#F92672">rax</font>
   <font color="#66D9EF">0x0000555555400c16</font> &lt;+48&gt;:	<font color="#A6E22E">mov</font><font color="#F8F8F2">    </font><font color="#F92672">eax</font>,<font color="#66D9EF">0x0</font>
   <font color="#66D9EF">0x0000555555400c1b</font> &lt;+53&gt;:	<font color="#A6E22E">call</font><font color="#F8F8F2">   </font><font color="#66D9EF">0x555555400920</font> &lt;<font color="#F92672">fprintf@plt</font>&gt;
   <font color="#66D9EF">0x0000555555400c20</font> &lt;+58&gt;:	<font color="#A6E22E">nop</font>
   <font color="#66D9EF">0x0000555555400c21</font> &lt;+59&gt;:	<font color="#A6E22E">pop</font><font color="#F8F8F2">    </font><font color="#F92672">rbp</font>
   <font color="#66D9EF">0x0000555555400c22</font> &lt;+60&gt;:	<font color="#A6E22E">ret</font><font color="#F8F8F2">    </font>
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

Having functions pointers mixed with data is interesting for us. If we can tamper with the a function pointer, it might give us the ability to call other functions.

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

0x555555603000	<font color="#F4BF75">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000291</font>	<font color="#A1EFE4">................</font>
0x555555603010	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000001</font>	<font color="#A1EFE4">................</font>
0x555555603020	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#A1EFE4">................</font>
[...]
0x555555603290	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#AE81FF">0x0000000000000061</font>	<font color="#AE81FF">........a.......</font>
0x5555556032a0	<font color="#AE81FF">0x0000000000000000</font>	<font color="#AE81FF">0x0000555555603010</font>	<font color="#AE81FF">.........0`UUU..</font>	 &lt;-- tcachebins[0x60][0/1]
0x5555556032b0	<font color="#AE81FF">0x6e6f53206568540a</font>	<font color="#AE81FF">0x7765726353206369</font>	<font color="#AE81FF">.The Sonic Screw</font>
0x5555556032c0	<font color="#AE81FF">0x0a0a726576697264</font>	<font color="#AE81FF">0x0a73726573616850</font>	<font color="#AE81FF">driver..Phasers.</font>
0x5555556032d0	<font color="#AE81FF">0x696f4e206568540a</font>	<font color="#AE81FF">0x6b63697243207973</font>	<font color="#AE81FF">.The Noisy Crick</font>
0x5555556032e0	<font color="#AE81FF">0x00000000000a7465</font>	<font color="#AE81FF">0x0000555555400be6</font>	<font color="#AE81FF">et........@UUU..</font>
0x5555556032f0	<font color="#AE81FF">0x0000000000000000</font>	<font color="#A6E22E">0x0000000000020d11</font>	<font color="#A6E22E">................</font>	 &lt;-- Top chu
</pre></code></pre></div>
{{< /rawhtml >}}



### Malloc()

The *3rd action* call the function `make_offer()` that **malloc** a memory of arbitrary size. And let us write data in this new *chunk*.

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


Again, let's do this in *gdb* an observe the memory with `vis_heap_chunks`.

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

0x555555603290	<font color="#F4BF75">0x0000000000000000</font>	<font color="#A1EFE4">0x0000000000000061</font>	<font color="#A1EFE4">........a.......</font>
0x5555556032a0	<font color="#A1EFE4">0x67694c206568540a</font>	<font color="#A1EFE4">0x0a72656261737468</font>	<font color="#A1EFE4">.The Lightsaber.</font>
0x5555556032b0	<font color="#A1EFE4">0x6e6f53206568540a</font>	<font color="#A1EFE4">0x7765726353206369</font>	<font color="#A1EFE4">.The Sonic Screw</font>
0x5555556032c0	<font color="#A1EFE4">0x0a0a726576697264</font>	<font color="#A1EFE4">0x0a73726573616850</font>	<font color="#A1EFE4">driver..Phasers.</font>
0x5555556032d0	<font color="#A1EFE4">0x696f4e206568540a</font>	<font color="#A1EFE4">0x6b63697243207973</font>	<font color="#A1EFE4">.The Noisy Crick</font>
0x5555556032e0	<font color="#A1EFE4">0x00000000000a7465</font>	<font color="#A1EFE4">0x0000555555400be6</font>	<font color="#A1EFE4">et........@UUU..</font>
0x5555556032f0	<font color="#A1EFE4">0x0000000000000000</font>	<font color="#AE81FF">0x0000000000000031</font>	<font color="#AE81FF">........1.......</font>
0x555555603300	<font color="#AE81FF">0x5959595959595959</font>	<font color="#AE81FF">0x5959595959595959</font>	<font color="#AE81FF">YYYYYYYYYYYYYYYY</font>
0x555555603310	<font color="#AE81FF">0x000000000000000a</font>	<font color="#AE81FF">0x0000000000000000</font>	<font color="#AE81FF">................</font>
0x555555603320	<font color="#AE81FF">0x0000000000000000</font>	<font color="#A6E22E">0x0000000000020ce1</font>	<font color="#A6E22E">................</font>	 &lt;-- Top chunk
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

The binary has a variable **`storage`** that holds some *ascii data* and a **pointer** to `printStorage()`. The binary also has a secret function `unlock_storage()` that gives us a shell.

* action **1** will **follow the pointer** in `storage` to call `printStorage()`. This prints the data of `storage`.
* action **3** *malloc()* an arbitrary size, and write data into this new memory.
* action **4** *free()* the variable `storage`.


>  If you haven't solved the binary yet. Stop here, and try to exploit the binary.


## The Bug

We have a **Use-After-Free** bug, which mean we can 




{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">
</code></pre></div>
{{< /rawhtml >}}
