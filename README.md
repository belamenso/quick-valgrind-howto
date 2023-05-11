# How to use `valgrind`

`valgrind` is a tool that makes debugging memory problems in C++ (memory leaks, double frees, segfaults) much easier.

You need a **Linux** machine for this. You were provided with EPFL virtual machines, if not, your Linux machine or WSL on Windows should be OK. Windows by itself it not OK. I'm aware of some people successfully using `valgrind` on macOS but I cannot help with that.

`valgrind` works on executables, so first you have to compile your code:

```sh
g++ -Wall -Wpedantic -g testX.cc -o testX
```

The `-g` flag is very important, without if you will not see information about files or lines on which you have problems. If you're using a Makefile, you want to append `-g` to the list of compiler options (usually called `CXXFLAGS`).

After this you have to open terminal in the directory of the executable `testX` and execute the following command:
```sh
valgrind --leak-check=full ./testX
```
## Behavior on everything OK
testX.cc:
```cpp
#include <iostream>
int main() {
	std::cout << "Hello\n";
}
```
`valgrind --leak-check=full ./testX`
```
==21255== Memcheck, a memory error detector
==21255== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21255== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==21255== Command: ./testX
==21255== 
Hello
==21255== 
==21255== HEAP SUMMARY:
==21255==     in use at exit: 0 bytes in 0 blocks
==21255==   total heap usage: 2 allocs, 2 frees, 73,728 bytes allocated
==21255== 
==21255== All heap blocks were freed -- no leaks are possible
==21255== 
==21255== For lists of detected and suppressed errors, rerun with: -s
==21255== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```
Note the "All heap blocks were freed -- no leaks are possible" line, this is what you want to see.
## Behavior on memory leak
testX.cc:
```cpp
int main() {
    new int;
}
```
`valgrind --leak-check=full ./testX`
```
==21118== Memcheck, a memory error detector
==21118== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21118== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==21118== Command: ./testX
==21118== 
==21118== 
==21118== HEAP SUMMARY:
==21118==     in use at exit: 4 bytes in 1 blocks
==21118==   total heap usage: 2 allocs, 1 frees, 72,708 bytes allocated
==21118== 
==21118== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==21118==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21118==    by 0x10915A: main (testX.cc:2)
==21118== 
==21118== LEAK SUMMARY:
==21118==    definitely lost: 4 bytes in 1 blocks
==21118==    indirectly lost: 0 bytes in 0 blocks
==21118==      possibly lost: 0 bytes in 0 blocks
==21118==    still reachable: 0 bytes in 0 blocks
==21118==         suppressed: 0 bytes in 0 blocks
==21118== 
==21118== For lists of detected and suppressed errors, rerun with: -s
==21118== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
As you can see, it showed you that you have unfreed memory on program exit and it points you to the place where you allocated this memory: `(testX.cc:2)` meaning file `testX.cc`, line 2.

## Behavior on double free
testX.cc:
```cpp
int main() {
    int *x = new int;
    delete x;
    delete x;
}
```
`valgrind --leak-check=full ./testX`
```
==21380== Memcheck, a memory error detector
==21380== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21380== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==21380== Command: ./testX
==21380== 
==21380== Invalid free() / delete / delete[] / realloc()
==21380==    at 0x484BB6F: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21380==    by 0x1091AE: main (testX.cc:4)
==21380==  Address 0x4de0c80 is 0 bytes inside a block of size 4 free'd
==21380==    at 0x484BB6F: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21380==    by 0x109198: main (testX.cc:3)
==21380==  Block was alloc'd at
==21380==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21380==    by 0x10917E: main (testX.cc:2)
==21380== 
==21380== 
==21380== HEAP SUMMARY:
==21380==     in use at exit: 0 bytes in 0 blocks
==21380==   total heap usage: 2 allocs, 3 frees, 72,708 bytes allocated
==21380== 
==21380== All heap blocks were freed -- no leaks are possible
==21380== 
==21380== For lists of detected and suppressed errors, rerun with: -s
==21380== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
It shows you that you attempted to free something at `(testX.cc:4)`, this thing was allocated at `(testX.cc:2)` and that you already freed it at `(testX.cc:3)`.


## Behavior on segfault (reading/writing memory you don't have)
testX.cc:
```cpp
int main() {
    int *x = new int;
    x += 1000000; // move the pointer 4 million bytes further than where it was previously
    *x += 1; // attempt to read and write to this new location
}
```
`valgrind --leak-check=full ./testX`
```
==21726== Memcheck, a memory error detector
==21726== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21726== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==21726== Command: ./testX
==21726== 
==21726== Invalid read of size 4
==21726==    at 0x10916F: main (testX.cc:4)
==21726==  Address 0x51b1580 is 3,999,920 bytes inside an unallocated block of size 4,121,360 in arena "client"
==21726== 
==21726== Invalid write of size 4
==21726==    at 0x109178: main (testX.cc:4)
==21726==  Address 0x51b1580 is 3,999,920 bytes inside an unallocated block of size 4,121,360 in arena "client"
==21726== 
==21726== 
==21726== HEAP SUMMARY:
==21726==     in use at exit: 4 bytes in 1 blocks
==21726==   total heap usage: 2 allocs, 1 frees, 72,708 bytes allocated
==21726== 
==21726== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==21726==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21726==    by 0x10915E: main (testX.cc:2)
==21726== 
==21726== LEAK SUMMARY:
==21726==    definitely lost: 4 bytes in 1 blocks
==21726==    indirectly lost: 0 bytes in 0 blocks
==21726==      possibly lost: 0 bytes in 0 blocks
==21726==    still reachable: 0 bytes in 0 blocks
==21726==         suppressed: 0 bytes in 0 blocks
==21726== 
==21726== For lists of detected and suppressed errors, rerun with: -s
==21726== ERROR SUMMARY: 3 errors from 3 contexts (suppressed: 0 from 0)
```
It tells you where exactly you performed an operation on memory that you don't have access to (read at `(testX.cc:4)` and write at `(testX.cc:4)`).


## Behavior on accessing memory that you already freed
testX.cc:
```cpp
int main() {
    int *x = new int;
    delete x;
    *x += 1;
}
```
`valgrind --leak-check=full ./testX`
```
==21834== Memcheck, a memory error detector
==21834== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21834== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==21834== Command: ./testX
==21834== 
==21834== Invalid read of size 4
==21834==    at 0x10919D: main (testX.cc:4)
==21834==  Address 0x4de0c80 is 0 bytes inside a block of size 4 free'd
==21834==    at 0x484BB6F: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21834==    by 0x109198: main (testX.cc:3)
==21834==  Block was alloc'd at
==21834==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21834==    by 0x10917E: main (testX.cc:2)
==21834== 
==21834== Invalid write of size 4
==21834==    at 0x1091A6: main (testX.cc:4)
==21834==  Address 0x4de0c80 is 0 bytes inside a block of size 4 free'd
==21834==    at 0x484BB6F: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21834==    by 0x109198: main (testX.cc:3)
==21834==  Block was alloc'd at
==21834==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==21834==    by 0x10917E: main (testX.cc:2)
==21834== 
==21834== 
==21834== HEAP SUMMARY:
==21834==     in use at exit: 0 bytes in 0 blocks
==21834==   total heap usage: 2 allocs, 2 frees, 72,708 bytes allocated
==21834== 
==21834== All heap blocks were freed -- no leaks are possible
==21834== 
==21834== For lists of detected and suppressed errors, rerun with: -s
==21834== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)

```
It tells you where exactly you performed an operation on memory (read at `(testX.cc:4)` and write at `(testX.cc:4)`) and that this piece of memory was allocated at `(testX.cc:2)` and already freed at `(testX.cc:3)`.
