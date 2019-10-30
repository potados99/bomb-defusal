# How To Beat Dr. Evil.

## Get Ready

### Retrive assembly from binary

~~~
 $ objdump -d bomb > dump.txt
~~~

### Get strings

~~~
 $ objdump -d bomd -j .rodata > strings.txt
~~~


## Solve

### Phase 1

 Bomb at phase 1 explodes if an input string does not match ((char *)0x402670).    
We can locate that string in rodata sector in the executable.
