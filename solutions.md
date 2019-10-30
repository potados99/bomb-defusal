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
  
  >>> Loop begin
  400f7e:	01 d2                	add    %edx,%edx				# Save 2 * %edx to %edx. %edx += %edx.
  400f80:	83 c0 01             	add    $0x1,%eax				# Add 1 to %eax.
  400f83:	39 d8                	cmp    %ebx,%eax				# Compare %ebx(value of 1) and %eax(value of >= 8).
  400f85:	75 f7                	jne    400f7e <phase_2+0x35>	# If %ebx != %eax, go to 0x400f7e.
  <<< Loop end

  The loop above finishes when %ebx reaches %eax.
  After the loop, %ebx will have a same value as %eax,
  and %edx will have value of %rsp << (%rax - %rbx).
  (The %edx is set, inside the <explode_bomb>, to the same value of %rsi, 
   which is also same as %rsp.)

  >>> Loop begin
  400f87:	03 55 fc             	add    -0x4(%rbp),%edx			# Add vars[0] to %edx.
  																	# The %rbp has value of %rsp + 4, so the %rbp - 4 means %rsp.
  400f8a:	39 55 00             	cmp    %edx,0x0(%rbp)			# Compare %edx and (%rbp).
  400f8d:	74 05                	je     400f94 <phase_2+0x4b>	# if %edx == (%rbp), keep going.
  400f8f:	e8 17 07 00 00       	callq  4016ab <explode_bomb>	# >> Or die.
  400f94:	83 c3 01             	add    $0x1,%ebx				# Add 1 to %ebx.
  400f97:	48 83 c5 04          	add    $0x4,%rbp				# Add 4 to %rbp.
  400f9b:	83 fb 07             	cmp    $0x7,%ebx				# Compare 7 and %ebx.
  400f9e:	74 10                	je     400fb0 <phase_2+0x67>	# >> If 7 == %ebx, go to the end of the function.
  400fa0:	ba 01 00 00 00       	mov    $0x1,%edx				# Save 1 to %edx.
  400fa5:	b8 00 00 00 00       	mov    $0x0,%eax				# Save 0 to %eax.
  400faa:	85 db                	test   %ebx,%ebx				# Test %ebx.
  400fac:	7f d0                	jg     400f7e <phase_2+0x35>	# If %ebx > 0, go to 0x400f7e and start loop.
  400fae:	eb d7                	jmp    400f87 <phase_2+0x3e>	# Go to 0x400f87.
  << Loop end

  400fb0:	48 8b 44 24 28       	mov    0x28(%rsp),%rax			# Ready to finish.
  400fc5:	48 83 c4 38          	add    $0x38,%rsp				# Deallocate stack and return.
~~~


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
