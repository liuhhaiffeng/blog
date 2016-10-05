---
title: c语言转汇编语言shell脚本
date: 2016-09-17 17:58:57
tags: ["c","shell","linux"]
---



最近在学习汇编，为了方便写了一个shell脚本，用来把c语言转换为汇编语言。很方便。

## shell脚本

```shell
#!/bin/bash  
if [  "$#" = "0" ] ; then
        echo 'help: c2asm FILENAME'  
        exit 1;
fi


if [ -f "$1" ] ; then
        echo 'check file ok!'  
else
        echo 'c2asm: file not exist'  
        exit 1;
fi
echo 'generating asm file...'  
gcc -O0 -S "$1"
tmp=$1
asmfile=${tmp%%.c}.s;

echo "asm file generated: $asmfile"  
echo '==================asm===================='  
cat "$asmfile" | grep -v '\.'
echo '==================c======================'  
cat "$1"
echo '==================END===================='  
echo 'done'  

```
## 使用效果

```shell
check file ok!
generating asm file...
asm file generated: main.s
==================asm====================
main:
	pushq	%rbp
	movq	%rsp, %rbp
	movl	$1, -8(%rbp)
	movl	$1, -4(%rbp)
	movl	-8(%rbp), %edx
	movl	-4(%rbp), %eax
	addl	%edx, %eax
	popq	%rbp
	ret
==================c======================

int main()
{
	int a,b;
	a = 1;
	b = 1;
	return a+b;

}
==================END====================
done


```





