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

 Bomb at phase 1 explodes if an input string does not match `((char *)0x402670)`.    
We can locate that string in rodata sector in the executable.

### Phase 2

 Given assembly of phase 2 is like below:

~~~
0000000000400f49 <phase_2>:
  400f49:	55                   	push   %rbp
  400f4a:	53                   	push   %rbx
  400f4b:	48 83 ec 38          	sub    $0x38,%rsp
  400f4f:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  400f56:	00 00 
  400f58:	48 89 44 24 28       	mov    %rax,0x28(%rsp)
  400f5d:	31 c0                	xor    %eax,%eax
  400f5f:	48 89 e6             	mov    %rsp,%rsi
  400f62:	e8 7a 07 00 00       	callq  4016e1 <read_numbers>
  400f67:	83 3c 24 00          	cmpl   $0x0,(%rsp)
  400f6b:	7f 05                	jg     400f72 <phase_2+0x29>
  400f6d:	e8 39 07 00 00       	callq  4016ab <explode_bomb>
  400f72:	48 8d 6c 24 04       	lea    0x4(%rsp),%rbp
  400f77:	bb 01 00 00 00       	mov    $0x1,%ebx
  400f7c:	eb 22                	jmp    400fa0 <phase_2+0x57>
  400f7e:	01 d2                	add    %edx,%edx
  400f80:	83 c0 01             	add    $0x1,%eax
  400f83:	39 d8                	cmp    %ebx,%eax
  400f85:	75 f7                	jne    400f7e <phase_2+0x35>
  400f87:	03 55 fc             	add    -0x4(%rbp),%edx
  400f8a:	39 55 00             	cmp    %edx,0x0(%rbp)
  400f8d:	74 05                	je     400f94 <phase_2+0x4b>
  400f8f:	e8 17 07 00 00       	callq  4016ab <explode_bomb>
  400f94:	83 c3 01             	add    $0x1,%ebx
  400f97:	48 83 c5 04          	add    $0x4,%rbp
  400f9b:	83 fb 07             	cmp    $0x7,%ebx
  400f9e:	74 10                	je     400fb0 <phase_2+0x67>
  400fa0:	ba 01 00 00 00       	mov    $0x1,%edx
  400fa5:	b8 00 00 00 00       	mov    $0x0,%eax
  400faa:	85 db                	test   %ebx,%ebx
  400fac:	7f d0                	jg     400f7e <phase_2+0x35>
  400fae:	eb d7                	jmp    400f87 <phase_2+0x3e>
  400fb0:	48 8b 44 24 28       	mov    0x28(%rsp),%rax
  400fb5:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  400fbc:	00 00 
  400fbe:	74 05                	je     400fc5 <phase_2+0x7c>
  400fc0:	e8 cb fb ff ff       	callq  400b90 <__stack_chk_fail@plt>
  400fc5:	48 83 c4 38          	add    $0x38,%rsp
  400fc9:	5b                   	pop    %rbx
  400fca:	5d                   	pop    %rbp
  400fcb:	c3                   	retq   
~~~

It's a bit complicated. Lets have a look at that.    
We don't need lines like `push   %rbp`, `push   %rbx` and `retq`, thus will eleminate them.
Also we will attach some comments.

Stack smashing detecting is applied to this function. 
Stack canary is saved at the beginning of the function, and checked at the end of the function.
We only need to figure out what this function does. So remove them also.

~~~
0000000000400f49 <phase_2>:
  400f4b:	48 83 ec 38          	sub    $0x38,%rsp				# Allocate 0x38(56) bytes in stack,
  																	  which means 7 of 8 byte variables.
  400f5d:	31 c0                	xor    %eax,%eax				# Set %eax to zero.
  400f5f:	48 89 e6             	mov    %rsp,%rsi				# Save %rsp to %rsi.
  400f62:	e8 7a 07 00 00       	callq  4016e1 <read_numbers>	# Call <read_numbers>. Pass the address of the array(may be) of 7 variables.
  400f67:	83 3c 24 00          	cmpl   $0x0,(%rsp)				# Compare 0 and (%rsp).
  400f6b:	7f 05                	jg     400f72 <phase_2+0x29>	# If vars[0] is over zero, keep going.
  400f6d:	e8 39 07 00 00       	callq  4016ab <explode_bomb>	# Or die.
  400f72:	48 8d 6c 24 04       	lea    0x4(%rsp),%rbp			# Save %rsp + 4 to %rbp.
  400f77:	bb 01 00 00 00       	mov    $0x1,%ebx				# Save 1 to %ebx.
  400f7c:	eb 22                	jmp    400fa0 <phase_2+0x57>	# Go to 0x400fa0.

  Set %edx to 1, %eax to 0 and start loop below if %ebx > 0.
  In this case, %ebx is just set to 1 above. So get into the loop.

  >>> Loop1 begin
  400f7e:	01 d2                	add    %edx,%edx				# Save 2 * %edx to %edx. %edx += %edx.
  400f80:	83 c0 01             	add    $0x1,%eax				# Add 1 to %eax.
  400f83:	39 d8                	cmp    %ebx,%eax				# Compare %ebx and %eax.
  400f85:	75 f7                	jne    400f7e <phase_2+0x35>	# If %ebx != %eax, go to 0x400f7e.
  <<< Loop1 end

  After the loop, %edx would be 1 << (%ebx - %eax).
  %ebx: 1
  %eax: 1
  %edx: 2

  400f87:	03 55 fc             	add    -0x4(%rbp),%edx			# Add vars[0] to %edx.
  																	  The %rbp has value of %rsp + 4, so the %rbp - 4 means %rsp.
  400f8a:	39 55 00             	cmp    %edx,0x0(%rbp)			# Compare %edx and (%rbp).
  400f8d:	74 05                	je     400f94 <phase_2+0x4b>	# if %edx == (%rbp), keep going.
  400f8f:	e8 17 07 00 00       	callq  4016ab <explode_bomb>	# >> Or die.
  400f94:	83 c3 01             	add    $0x1,%ebx				# Add 1 to %ebx. Now 2.
  400f97:	48 83 c5 04          	add    $0x4,%rbp				# Add 4 to %rbp. Now points &vars[1]. 
  400f9b:	83 fb 07             	cmp    $0x7,%ebx				# Compare 7 and %ebx.
  400f9e:	74 10                	je     400fb0 <phase_2+0x67>	# >> If 7 == %ebx, go to the end of the function.
  400fa0:	ba 01 00 00 00       	mov    $0x1,%edx				# Save 1 to %edx.
  400fa5:	b8 00 00 00 00       	mov    $0x0,%eax				# Save 0 to %eax.
  400faa:	85 db                	test   %ebx,%ebx				# Test %ebx.
  400fac:	7f d0                	jg     400f7e <phase_2+0x35>	# If %ebx > 0, go to 0x400f7e and start loop.
  400fae:	eb d7                	jmp    400f87 <phase_2+0x3e>	# Go to 0x400f87.
  400fb0:	48 8b 44 24 28       	mov    0x28(%rsp),%rax			# Ready to finish.
  400fc5:	48 83 c4 38          	add    $0x38,%rsp				# Deallocate stack and return.
~~~

Phase 2 calls <read_numbers>, which read seven numbers from user input.

~~~
00000000004016e1 <read_numbers>:
  4016e1:	48 83 ec 10          	sub    $0x10,%rsp					# Allocated 0x10(16) bytes in stack.
  4016e5:	48 89 f2             	mov    %rsi,%rdx					# The pointer given. Phase 2 gives a stack pointer.
  4016e8:	48 8d 4e 04          	lea    0x4(%rsi),%rcx				# Save p(param) + 0x4(4) to %rcx.
  4016ec:	48 8d 46 18          	lea    0x18(%rsi),%rax				# Save p + 0x18(24) to %rax.
  4016f0:	50                   	push   %rax							# Save %rax(value of p + 0x18) to stack. %rsp -= 8.
  4016f1:	48 8d 46 14          	lea    0x14(%rsi),%rax				# Save p + 0x14(20) to %rax.
  4016f5:	50                   	push   %rax							# Save %rax(value of p + 0x14) to stack. %rsp -= 8.
  4016f6:	48 8d 46 10          	lea    0x10(%rsi),%rax				# Save p + 0x10(16) to %rax.
  4016fa:	50                   	push   %rax							# Save %rax(value of p + 0x10) to stack. %rsp -= 8.
  4016fb:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9				# Save p + 0xc(11) to %r9.
  4016ff:	4c 8d 46 08          	lea    0x8(%rsi),%r8				# Save p + 0x8(8) to %r8.
  401703:	be a1 29 40 00       	mov    $0x4029a1,%esi				# Save address of "%d %d %d %d %d %d %d" to %esi.
  401708:	b8 00 00 00 00       	mov    $0x0,%eax					# Save zero to %eax.
  40170d:	e8 2e f5 ff ff       	callq  400c40 <__isoc99_sscanf@plt> # Call ssanf. It takes >= 3 parameters:
  																		  in file, format string, addresses of variables to store.
																		  They are passed to %rdi, %rsi, %rdx and ....
																		  The first argument might be the INFILE, which is declared
																		  as a global variable.
																		  The second one is the format string.
																		  Currently the INFILE is in %rdi, format string is in %rsi.
  401712:	48 83 c4 20          	add    $0x20,%rsp
  401716:	83 f8 06             	cmp    $0x6,%eax					# Compare 0x6 and %eax.
  401719:	7f 05                	jg     401720 <read_numbers+0x3f>	# If returned value over 6(at least 7.), safely finish function.
  40171b:	e8 8b ff ff ff       	callq  4016ab <explode_bomb>		# Or explode bomb.
  401720:	48 83 c4 08          	add    $0x8,%rsp
  401724:	c3                   	retq   
~~~

%ebx and one of seven integer is compared sequencially.

The %ebx gets from 2 up to 128(2^7).

Total 7 compare, each input number should be 2^(index of number + 1).

Summary: 

- vars[0] must be over zero.
- vars[0] must be 2.
- vars[1] must be vars[0] + 2.
- vars[2] must be vars[1] + vars[1].
- vars[3] must be vars[2] + vars[2].

.
.
.

answer is 2 4 8 16 32 64 128.    

### Phase 3

 The assembly:

~~~
0000000000400fcc <phase_3>:
  400fcc:	48 83 ec 18          	sub    $0x18,%rsp						# Allocate 0x18(24) bytes in stack.
  400fd0:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax					# Save stack canary to %rax.
  400fd7:	00 00 
  400fd9:	48 89 44 24 08       	mov    %rax,0x8(%rsp)					# Save stack canary to stack with offset +8.
  400fde:	31 c0                	xor    %eax,%eax						# Clear %rax.
  400fe0:	48 8d 4c 24 04       	lea    0x4(%rsp),%rcx					# Save %rsp + 4 to %rcx.
  400fe5:	48 89 e2             	mov    %rsp,%rdx						# Save %rsp to %rdx.
  400fe8:	be b0 29 40 00       	mov    $0x4029b0,%esi					# Save address of "%d %d" to %rsi.
  400fed:	e8 4e fc ff ff       	callq  400c40 <__isoc99_sscanf@plt>		# Call scanf. Params are %rdi, %rsi, %rdx. %rsi is "%d %d", %rsi is %rsp.
  400ff2:	83 f8 01             	cmp    $0x1,%eax						# Compare returned value.
  400ff5:	7f 05                	jg     400ffc <phase_3+0x30>			# If over two items, keep going.
  400ff7:	e8 af 06 00 00       	callq  4016ab <explode_bomb>			# Or explode.
  400ffc:	8b 04 24             	mov    (%rsp),%eax						# Save input[0] to %eax.
 
  Below is switch-case satement.
  Indices range from 0 to 7, 
  so the input[0] must be in range from 43 to 50.

  400fff:	83 e8 2b             	sub    $0x2b,%eax						# Subtract 0x2b(43) from $eax.
  401002:	83 f8 07             	cmp    $0x7,%eax						# Compare unsigned %eax with 7.
  401005:	77 64                	ja     40106b <phase_3+0x9f>			# If over 7, jump and explode.
  401007:	89 c0                	mov    %eax,%eax						# NOP
  401009:	ff 24 c5 e0 26 40 00 	jmpq   *0x4026e0(,%rax,8)				# Jump! This is a switch-case statement!

  401010:	b8 88 00 00 00       	mov    $0x88,%eax						# >> 0
  401015:	eb 05                	jmp    40101c <phase_3+0x50>
  
  401017:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 1
  40101c:	2d 4c 02 00 00       	sub    $0x24c,%eax
  
  401021:	eb 05                	jmp    401028 <phase_3+0x5c>
  401023:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 2
  401028:	05 e9 02 00 00       	add    $0x2e9,%eax
  40102d:	eb 05                	jmp    401034 <phase_3+0x68>

  40102f:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 3
  401034:	2d 00 02 00 00       	sub    $0x200,%eax
  401039:	eb 05                	jmp    401040 <phase_3+0x74>
  
  40103b:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 4
  401040:	05 00 02 00 00       	add    $0x200,%eax
  401045:	eb 05                	jmp    40104c <phase_3+0x80>

  401047:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 5
  40104c:	2d 00 02 00 00       	sub    $0x200,%eax
  401051:	eb 05                	jmp    401058 <phase_3+0x8c>
  
  401053:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 6
  401058:	05 00 02 00 00       	add    $0x200,%eax
  40105d:	eb 05                	jmp    401064 <phase_3+0x98>
  
  40105f:	b8 00 00 00 00       	mov    $0x0,%eax						# >> 7
  401064:	2d 00 02 00 00       	sub    $0x200,%eax
  401069:	eb 0a                	jmp    401075 <phase_3+0xa9>
  40106b:	e8 3b 06 00 00       	callq  4016ab <explode_bomb>
  401070:	b8 00 00 00 00       	mov    $0x0,%eax						# Save 0 to %eax.
  401075:	3b 44 24 04          	cmp    0x4(%rsp),%eax					# Compare %eax with input[1].
  401079:	74 05                	je     401080 <phase_3+0xb4>			# If input[1] == 0, safe.
  40107b:	e8 2b 06 00 00       	callq  4016ab <explode_bomb>			# Or die.
  401080:	48 8b 44 24 08       	mov    0x8(%rsp),%rax					# Restore stack canary.
  401085:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax					# Compare.
~~~

The jump table look like:

~~~
(gdb) x/8gx 0x4026e0
0x4026e0:	0x0000000000401010	0x0000000000401017
0x4026f0:	0x0000000000401023	0x000000000040102f
0x402700:	0x000000000040103b	0x0000000000401047
0x402710:	0x0000000000401053	0x000000000040105f
~~~

Possible answers:

- 43 -219
 Route from 0.

- 44 -355
 Route from 1.

- 45 233
 Route from 2.

- 46 512
 Route from 3.

- 47 0
 Route from 4.

- 50 -512
 Route from 7.

### Phase 4

 Let's see.

~~~
00000000004010d5 <phase_4>:
  4010d5:	48 83 ec 18          	sub    $0x18,%rsp						# Allocate 24 bytes.
  4010d9:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax					# Save stack canary to %rax.
  4010e0:	00 00 
  4010e2:	48 89 44 24 08       	mov    %rax,0x8(%rsp)					# Save %rax(value of stack canary) to (%rsp + 8).
  4010e7:	31 c0                	xor    %eax,%eax						# Clear %rax.
  4010e9:	48 89 e1             	mov    %rsp,%rcx						# Save stack pointer to %rcx. &input[1].
  4010ec:	48 8d 54 24 04       	lea    0x4(%rsp),%rdx					# Save stack pointer + 4 to %rdx. &input[0].
  4010f1:	be b0 29 40 00       	mov    $0x4029b0,%esi					# Save address of "%d %d" to %rsi.
  4010f6:	e8 45 fb ff ff       	callq  400c40 <__isoc99_sscanf@plt>		# Call sscanf. Read two integers. Input numbers from %rsp + 4.
  4010fb:	83 f8 02             	cmp    $0x2,%eax						# Compare %rax with 2.
  4010fe:	75 0b                	jne    40110b <phase_4+0x36>			# If %rax != 2, jump and explode.
  401100:	8b 04 24             	mov    (%rsp),%eax						# Save (%rsp) to %rax.
  401103:	83 e8 03             	sub    $0x3,%eax						# %rax -= 3.
  401106:	83 f8 02             	cmp    $0x2,%eax						# Compare %rax with 2.
  401109:	76 05                	jbe    401110 <phase_4+0x3b>			# If %rax <= 2, keep going.
  40110b:	e8 9b 05 00 00       	callq  4016ab <explode_bomb>			# Or explode.
  401110:	8b 34 24             	mov    (%rsp),%esi						# Save (%rsp)(input[1]) to %rsi.
  401113:	bf 08 00 00 00       	mov    $0x8,%edi						# Save 8 to %rdi.
  401118:	e8 7d ff ff ff       	callq  40109a <func4>					# Call func4. Params: 8, *%rs0.
  40111d:	3b 44 24 04          	cmp    0x4(%rsp),%eax					# Compare %rax and (%rsp + 4)(input[0])
  401121:	74 05                	je     401128 <phase_4+0x53>			# If %rax == input[0], keep going.
  401123:	e8 83 05 00 00       	callq  4016ab <explode_bomb>			# Or explode.
  401128:	48 8b 44 24 08       	mov    0x8(%rsp),%rax					# Restore stack canary.
  40112d:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax					# Check canary.
  401134:	00 00 
  401136:	74 05                	je     40113d <phase_4+0x68>			# If ok, finish function.
  401138:	e8 53 fa ff ff       	callq  400b90 <__stack_chk_fail@plt>
  40113d:	48 83 c4 18          	add    $0x18,%rsp
  401141:	c3     
~~~


The func4 is like below:

~~~

000000000040109a <func4>:
  40109a:	85 ff                	test   %edi,%edi				# Test %rdi.
  40109c:	7e 2b                	jle    4010c9 <func4+0x2f>		# Go to 0x4010c9 if %rdi <= 0.
  40109e:	89 f0                	mov    %esi,%eax				# Save %rsi to %rax.
  4010a0:	83 ff 01             	cmp    $0x1,%edi				# Compare %rdi with 1.
  4010a3:	74 2e                	je     4010d3 <func4+0x39>		# If %rdi == 1, finish function.
  4010a5:	41 54                	push   %r12						# Save %r12 to stack.
  4010a7:	55                   	push   %rbp						# Save %rbp to stack.
  4010a8:	53                   	push   %rbx						# Save %rbx to stack.
  4010a9:	89 f5                	mov    %esi,%ebp				# Save %rsi to %rbp.
  4010ab:	89 fb                	mov    %edi,%ebx				# Save %rdi to %rbx.
  4010ad:	8d 7f ff             	lea    -0x1(%rdi),%edi			# Save %rdi - 1 to %rdi.
  4010b0:	e8 e5 ff ff ff       	callq  40109a <func4>			# Call itself!
  4010b5:	44 8d 64 05 00       	lea    0x0(%rbp,%rax,1),%r12d
  4010ba:	8d 7b fe             	lea    -0x2(%rbx),%edi
  4010bd:	89 ee                	mov    %ebp,%esi
  4010bf:	e8 d6 ff ff ff       	callq  40109a <func4>
  4010c4:	44 01 e0             	add    %r12d,%eax
  4010c7:	eb 06                	jmp    4010cf <func4+0x35>
  4010c9:	b8 00 00 00 00       	mov    $0x0,%eax
  4010ce:	c3                   	retq   
  4010cf:	5b                   	pop    %rbx
  4010d0:	5d                   	pop    %rbp
  4010d1:	41 5c                	pop    %r12
  4010d3:	f3 c3                	repz retq 

~~~

The function fun4 is recursive, making it harder to read.

So weask gdb for help.

~~~
(gdb) call func4(0, 1)
$1 = 0
(gdb) call func4(1, 1)
$2 = 1
(gdb) call func4(2, 1)
$3 = 2
(gdb) call func4(3, 1)
$4 = 4
(gdb) call func4(4, 1)
$5 = 7
(gdb) call func4(5, 1)
$6 = 12
(gdb) call func4(6, 1)
$7 = 20
(gdb) call func4(7, 1)
$8 = 33
(gdb) call func4(8, 1)
$9 = 54
(gdb) call func4(9, 1)
$10 = 88
(gdb) call func4(10, 1)
$11 = 143
~~~

Gocha!

fun4(x, y) = (Fibonacci(x + 1) - 1) * y.

In phase 4, the x is fixed to 8, so fun4(8, y) is 54 * y.


Summary:

- input[0] must be <= 5.
- input[0] must be >= 3.
- input[1] must be input[0] * 54.

Possible answers:

- 162 3
- 216 4
- 270 5

### Phase 5

~~~
0000000000401142 <phase_5>:
  401142:	53                   	push   %rbx						# Save %rbx to stack.
  401143:	48 89 fb             	mov    %rdi,%rbx				# Save %rdi to %rbx. In this case it is the address of input string.
  401146:	e8 6e 02 00 00       	callq  4013b9 <string_length>	# Measure string length.
  40114b:	83 f8 06             	cmp    $0x6,%eax				# Compare returned value with 6.
  40114e:	74 05                	je     401155 <phase_5+0x13>	# If strlen == 6, keep going.
  401150:	e8 56 05 00 00       	callq  4016ab <explode_bomb>	# Or explode.
  401155:	48 89 d8             	mov    %rbx,%rax				# Save address of input string to %rax.
  401158:	48 8d 7b 06          	lea    0x6(%rbx),%rdi			# Save address of input string + 1 to %rdi.
  40115c:	b9 00 00 00 00       	mov    $0x0,%ecx				# Save 0 to %ecx.

  Until the end of the string, add *(0x402720 + (4 * (str[i] & 0xf)))
  Total of them must be 0x21(33).

  >>> Loop1 begin
  401161:	0f b6 10             	movzbl (%rax),%edx				# Save currently pointing character of input string to %rdx.
  401164:	83 e2 0f             	and    $0xf,%edx				# Leave only nibble..
  401167:	03 0c 95 20 27 40 00 	add    0x402720(,%rdx,4),%ecx	# Add 0x402720(,%rdx,4) to %rcx.
  40116e:	48 83 c0 01          	add    $0x1,%rax				# Add 1 to %rax.
  401172:	48 39 f8             	cmp    %rdi,%rax				# Compare %rax with %rdi.
  401175:	75 ea                	jne    401161 <phase_5+0x1f>	# If %rax != %rdi, go to loop begin.
  <<< Loop1 end

  401177:	83 f9 21             	cmp    $0x21,%ecx				# Compare %rcx with 0x21(33).
  40117a:	74 05                	je     401181 <phase_5+0x3f>	# If %ecx == 0x21(33), finish function.
  40117c:	e8 2a 05 00 00       	callq  4016ab <explode_bomb>
  401181:	5b                   	pop    %rbx
  401182:	c3                   	retq   
~~~

The table at 0x402720 is like below:

~~~
  402720:	02 00 00 00 0a 00 00 00 06 00 00 00 01 00 00 00     ................
  402730:	0c 00 00 00 10 00 00 00 09 00 00 00 03 00 00 00     ................
  402740:	04 00 00 00 07 00 00 00 0e 00 00 00 05 00 00 00     ................
  402750:	0b 00 00 00 08 00 00 00 0f 00 00 00 0d 00 00 00     ................

  When n is str[i] & 0xf, table of n is like below:

  n		word

  0x0:	0x02
  0x1:	0x0a
  0x2:	0x06
  0x3:	0x01

  0x4:	0x0c
  0x5:	0x10
  0x6:	0x09
  0x7:	0x03

  0x8:	0x04
  0x9:	0x07
  0xa:	0x0e
  0xb:	0x05

  0xc:	0x0b
  0xd:	0x08
  0xe:	0x0f
  0xf:	0x0d
~~~

10 + 2 + 6 + 4 + 7 + 4

0 / 1 / 2 / 8 / 8 / 9

Summary:

- Length of input string must be 6.
- input_string.map(char -> n_to_word_table(char & 0f)).sum() must be 0x21(33).

Possible answers:

- Too many.
- One of them is pabxxi.
