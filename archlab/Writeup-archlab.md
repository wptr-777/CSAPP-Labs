# Arch lab

用Y86-64汇编去实现writeup.pdf里的功能

## Part A

用Y86汇编实现example.c中的函数

```
1 /* linked list element */
2 typedef struct ELE {
3 long val;
4 struct ELE *next;
5 } *list_ptr;
6
7 /* sum_list - Sum the elements of a linked list */
8 long sum_list(list_ptr ls)
9 {
10 long val = 0;
11 while (ls) {
12 val += ls->val;
13 ls = ls->next;
14 }
15 return val;
16 }
17
18 /* rsum_list - Recursive version of sum_list */
19 long rsum_list(list_ptr ls)
20 {
21 if (!ls)
22 return 0;
23 else {
24 long val = ls->val;
25 long rest = rsum_list(ls->next);
26 return val + rest;
27 }
28 }
29
30 /* copy_block - Copy src to dest and return xor checksum of src */
31 long copy_block(long *src, long *dest, long len)
32 {
33 long result = 0;
34 while (len > 0) {
35 long val = *src++;
36 *dest++ = val;
 result ˆ= val;
 len--;
 }
 return result; 
}
```

sum.ys

```
.pos 0
init: irmovq Stack, %rsp
      call Main
      halt

        .align 8
ele1:
        .quad 0x00a
        .quad ele2
ele2:
        .quad 0x0b0
        .quad ele3
ele3:
        .quad 0xc00
        .quad 0

Main:
        irmovq ele1,%rdi
        call sum_list
        ret

sum_list:
        irmovq $0,%rax
        jmp test
loop:
        mrmovq 0(%rdi),%rcx
        addq %rcx,%rax
        mrmovq 8(%rdi),%rcx
        rrmovq %rcx,%rdi
test:
        andq %rdi,%rdi
        jne loop
        ret

        .pos 0x100
Stack:
```

rsum.ys

```assembly

	.pos 0
	irmovq stack, %rsp
	call main
	halt

# Sample linked list
	    .align 8
	    ele1:
	    .quad 0x00a
	    .quad e1e2
	    e1e2:
	    .quad 0x0b0
	    .quad e1e3
	    e1e3:
	    .quad 0xc00
	    .quad 0

main:	irmovq ele1, %rdi
	    call rsum_list
	    ret


rsum_list:
	    pushq %r12
	    irmovq $0, %rax
	    andq %rdi,%rdi
	    je re
	    mrmovq 0(%rdi), %r12
	    mrmovq 8(%rdi), %rdi
	    call rsum_list
	    addq %r12, %rax
re:	    popq %r12
	    ret

	    .pos 0x100
stack:

```

copy.ys

```assembly

	.pos 0
	irmovq stack, %rsp
	call main
	halt

	.align 8
# Source block
src:
	.quad 0x00a
	.quad 0x0b0
	.quad 0xc00

# Destination block
dest:
	.quad 0x111
	.quad 0x222
	.quad 0x333

main:
	irmovq src, %rdi
	irmovq dest, %rsi
	irmovq $3, %rdx
	call copy_block
	ret

copy_block:
	pushq %r12
	pushq %r13
	pushq %r14
	irmovq $1, %r13
	irmovq $8, %r14
	irmovq $0, %rax 
	jmp Tloop
loop:
	mrmovq 0(%rdi), %r12 
	addq %r14, %rdi
	rmmovq %r12, (%rsi)
	addq %r14, %rsi
	xorq %r12, %rax
	subq %r13, %rdx
Tloop:
	andq %rdx, %rdx
	jg loop
	popq %r14
	popq %r13
	popq %r12
	ret

	.pos 0x100
stack:

```

## Part B

考察hcl指令

先修改一下MakeFile才能正常编译，我找不到tk和tcl，在MakeFile里注释掉一些语句

在seq-full.hcl中修改些东西就可以了

我更改后的seq-full.hcl

```
#/* $begin seq-all-hcl */
####################################################################
#  HCL Description of Control for Single Cycle Y86-64 Processor SEQ   #
#  Copyright (C) Randal E. Bryant, David R. O'Hallaron, 2010       #
####################################################################

## Your task is to implement the iaddq instruction
## The file contains a declaration of the icodes
## for iaddq (IIADDQ)
## Your job is to add the rest of the logic to make it work

####################################################################
#    C Include's.  Don't alter these                               #
####################################################################

quote '#include <stdio.h>'
quote '#include "isa.h"'
quote '#include "sim.h"'
quote 'int sim_main(int argc, char *argv[]);'
quote 'word_t gen_pc(){return 0;}'
quote 'int main(int argc, char *argv[])'
quote '  {plusmode=0;return sim_main(argc,argv);}'

####################################################################
#    Declarations.  Do not change/remove/delete any of these       #
####################################################################

##### Symbolic representation of Y86-64 Instruction Codes #############
wordsig INOP 	'I_NOP'
wordsig IHALT	'I_HALT'
wordsig IRRMOVQ	'I_RRMOVQ'
wordsig IIRMOVQ	'I_IRMOVQ'
wordsig IRMMOVQ	'I_RMMOVQ'
wordsig IMRMOVQ	'I_MRMOVQ'
wordsig IOPQ	'I_ALU'
wordsig IJXX	'I_JMP'
wordsig ICALL	'I_CALL'
wordsig IRET	'I_RET'
wordsig IPUSHQ	'I_PUSHQ'
wordsig IPOPQ	'I_POPQ'
# Instruction code for iaddq instruction
wordsig IIADDQ	'I_IADDQ'

##### Symbolic represenations of Y86-64 function codes                  #####
wordsig FNONE    'F_NONE'        # Default function code

##### Symbolic representation of Y86-64 Registers referenced explicitly #####
wordsig RRSP     'REG_RSP'    	# Stack Pointer
wordsig RNONE    'REG_NONE'   	# Special value indicating "no register"

##### ALU Functions referenced explicitly                            #####
wordsig ALUADD	'A_ADD'		# ALU should add its arguments

##### Possible instruction status values                             #####
wordsig SAOK	'STAT_AOK'	# Normal execution
wordsig SADR	'STAT_ADR'	# Invalid memory address
wordsig SINS	'STAT_INS'	# Invalid instruction
wordsig SHLT	'STAT_HLT'	# Halt instruction encountered

##### Signals that can be referenced by control logic ####################

##### Fetch stage inputs		#####
wordsig pc 'pc'				# Program counter
##### Fetch stage computations		#####
wordsig imem_icode 'imem_icode'		# icode field from instruction memory
wordsig imem_ifun  'imem_ifun' 		# ifun field from instruction memory
wordsig icode	  'icode'		# Instruction control code
wordsig ifun	  'ifun'		# Instruction function
wordsig rA	  'ra'			# rA field from instruction
wordsig rB	  'rb'			# rB field from instruction
wordsig valC	  'valc'		# Constant from instruction
wordsig valP	  'valp'		# Address of following instruction
boolsig imem_error 'imem_error'		# Error signal from instruction memory
boolsig instr_valid 'instr_valid'	# Is fetched instruction valid?

##### Decode stage computations		#####
wordsig valA	'vala'			# Value from register A port
wordsig valB	'valb'			# Value from register B port

##### Execute stage computations	#####
wordsig valE	'vale'			# Value computed by ALU
boolsig Cnd	'cond'			# Branch test

##### Memory stage computations		#####
wordsig valM	'valm'			# Value read from memory
boolsig dmem_error 'dmem_error'		# Error signal from data memory


####################################################################
#    Control Signal Definitions.                                   #
####################################################################

################ Fetch Stage     ###################################

# Determine instruction code
word icode = [
	imem_error: INOP;
	1: imem_icode;		# Default: get from instruction memory
];

# Determine instruction function
word ifun = [
	imem_error: FNONE;
	1: imem_ifun;		# Default: get from instruction memory
];

bool instr_valid = icode in 
	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
	       IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };

# Does fetched instruction require a regid byte?
bool need_regids =
	icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };

# Does fetched instruction require a constant word?
bool need_valC =
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };

################ Decode Stage    ###################################

## What register should be used as the A source?
word srcA = [
	icode in { IRRMOVQ, IRMMOVQ, IOPQ, IPUSHQ  } : rA;
	icode in { IPOPQ, IRET } : RRSP;
	1 : RNONE; # Don't need register
];

## What register should be used as the B source?
word srcB = [
	icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ  } : rB;
	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
	1 : RNONE;  # Don't need register
];

## What register should be used as the E destination?
word dstE = [
	icode in { IRRMOVQ } && Cnd : rB;
	icode in { IIRMOVQ, IOPQ, IIADDQ} : rB;
	icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
	1 : RNONE;  # Don't write any register
];

## What register should be used as the M destination?
word dstM = [
	icode in { IMRMOVQ, IPOPQ } : rA;
	1 : RNONE;  # Don't write any register
];

################ Execute Stage   ###################################

## Select input A to ALU
word aluA = [
	icode in { IRRMOVQ, IOPQ } : valA;
	icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : valC;
	icode in { ICALL, IPUSHQ } : -8;
	icode in { IRET, IPOPQ } : 8;
	# Other instructions don't need ALU
];

## Select input B to ALU
word aluB = [
	icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
		      IPUSHQ, IRET, IPOPQ, IIADDQ } : valB;
	icode in { IRRMOVQ, IIRMOVQ } : 0;
	# Other instructions don't need ALU
];

## Set the ALU function
word alufun = [
	icode == IOPQ : ifun;
	1 : ALUADD;
];

## Should the condition codes be updated?
+bool set_cc = icode in { IOPQ, IIADDQ };

################ Memory Stage    ###################################

## Set read control signal
bool mem_read = icode in { IMRMOVQ, IPOPQ, IRET };

## Set write control signal
bool mem_write = icode in { IRMMOVQ, IPUSHQ, ICALL };

## Select memory address
word mem_addr = [
	icode in { IRMMOVQ, IPUSHQ, ICALL, IMRMOVQ } : valE;
	icode in { IPOPQ, IRET } : valA;
	# Other instructions don't need address
];

## Select memory input data
word mem_data = [
	# Value from register
	icode in { IRMMOVQ, IPUSHQ } : valA;
	# Return PC
	icode == ICALL : valP;
	# Default: Don't write anything
];

## Determine instruction status
word Stat = [
	imem_error || dmem_error : SADR;
	!instr_valid: SINS;
	icode == IHALT : SHLT;
	1 : SAOK;
];

################ Program Counter Update ############################

## What address should instruction be fetched at

word new_pc = [
	# Call.  Use instruction constant
	icode == ICALL : valC;
	# Taken branch.  Use instruction constant
	icode == IJXX && Cnd : valC;
	# Completion of RET instruction.  Use value from stack
	icode == IRET : valM;
	# Default: Use incremented PC
	1 : valP;
];
#/* $end seq-all-hcl */
```

最后执行命令验证成果即可

```shell
(cd ../ptest; make SIM=../seq/ssim TFLAGS=-i)
```

## Part C

优化函数，修改pipe-full.hcl和ncopy.ys来改进函数性能

当然也可以用先在pipe-line.hcl中实现iaddl指令

不过我直接开撸了，我的答案如下

ncopy.ys

```assembly
# You can modify this portion
	# Loop header
	xorq %rax,%rax		# count = 0;
	iaddq $-4, %rdx
	jle EQ0

Npos0:
	mrmovq (%rdi), %r10
	mrmovq 8(%rdi), %r11
	mrmovq 16(%rdi), %r12
	mrmovq 24(%rdi), %r13
	mrmovq 32(%rdi), %r14
	rmmovq %r10, (%rsi)
	andq %r10, %r10		# val <= 0?
    jle Npos1
    iaddq $1, %rax

Npos1:
	rmmovq %r11, 8(%rsi)
	andq %r11, %r11		# val <= 0?
    jle Npos2
    iaddq $1, %rax

Npos2:
	rmmovq %r12, 16(%rsi)
	andq %r12, %r12		# val <= 0?
    jle Npos3
    iaddq $1, %rax

Npos3:
	rmmovq %r13, 24(%rsi)
	andq %r13, %r13	# val <= 0?
    jle Npos4
    iaddq $1, %rax

Npos4:
	rmmovq %r14, 32(%rsi)
	andq %r14, %r14	# val <= 0?
    jle Tail
    iaddq $1, %rax

Tail:
	iaddq $40, %rsi
	iaddq $40, %rdi
	iaddq $-5, %rdx
    jg Npos0

EQ0:
    iaddq $4, %rdx
    jle Done
    mrmovq (%rdi), %r10
    mrmovq 8(%rdi), %r11
    rmmovq %r10, (%rsi)
    andq %r10, %r10
    jle EQ1
    iaddq $1, %rax

EQ1:
    iaddq $-1, %rdx
    jle Done
    rmmovq %r11, 8(%rsi)
    andq %r11, %r11
    jle EQ2
    iaddq $1, %rax

EQ2:
    iaddq $-1, %rdx
    jle Done
    mrmovq 16(%rdi), %r12
    rmmovq %r12, 16(%rsi)
    andq %r12, %r12
    jle EQ3
    iaddq $1, %rax

EQ3:
    iaddq $-1, %rdx
    jle Done
    mrmovq 24(%rdi), %r13
    rmmovq %r13, 24(%rsi)
    andq %r13, %r13
    jle Done
    iaddq $1, %rax

```

最后

```shell
./correctness.pl #检验是否正确
./benchmark.pl #检验性能
```

