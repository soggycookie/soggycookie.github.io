---
layout: post
title: A quest to understand how virtual work in c++
---

  After reading some other authors' articles about how vtable and inheritance works, I found out I understand nothing. So I took this matter into my own hands, and I also thought that this is a good chance to learn more about other tools than Visual Studio toolchain. *Mingw64* toolchain was selected to get myself used to cmd tool.

## **Case 1:** No inheritance


```
#include <iostream>
class Base
{
public:
    Base() = default;


     ~Base() = default;


     void Foo()
    {
        std::cout << "Base" << std::endl;
    }
};


class Derived
{
public:
    Derived() = default;


    void Foo()
    {
        std::cout << "Derived" << std::endl;
    }    
};


int main()
{
    std::cout << sizeof(Base) << std::endl;
    std::cout << sizeof(Derived) << std::endl;    
    base->Foo();
    derived->Foo();




    delete base;
    delete derived;
    return 0;
}

```

Output of the PE: 
```
1
1
Base
Derived
```

  The size of both classes are 1 when no data or virtual functions are present. C++ guarantees that every object has a unique address, so the compiler allocates at least 1 byte for such empty objects.

  Now let’s see how these functions are invoked in Assembly.
```
 # no_virtual.cpp:35:     base->Foo();
	mov	rax, QWORD PTR -8[rbp]	 # tmp116, base
	mov	rcx, rax	 #, tmp116
	call	_ZN4Base3FooEv	 #
 # no_virtual.cpp:36:     derived->Foo();
	mov	rax, QWORD PTR -16[rbp]	 # tmp117, derived
	mov	rcx, rax	 #, tmp117
	call	_ZN7Derived3FooEv	 #
```

  The first two mov instructions load the object pointer (*this*) from the stack frame and place it into *rcx* register, which is the first argument register according to the Windows x64 calling convention.

  The call instructions directly invoke functions whose targets are known at compile time and resolved at link time, resulting in direct (non-virtual) calls to *Base::Foo()* and *Derived::Foo()*. No vtable lookup or indirect jump is involved (See later).

  One more cool thing is how we fetch local variables from the stack. Local variables are accessed via negative offsets from the frame pointer (*rbp*), such as -8 and -16, which reflects the fact that the x64 stack grows toward lower memory addresses (from higher to lower addresses).

## **Case 2:** Simple inheritance

```
#include <iostream>
class Base
{
public:
    Base() = default;
    virtual ~Base() = default;
    virtual void Foo()
    {
        std::cout << "Base" << std::endl;
    }
};


class Derived : public Base
{
public:
    Derived() = default;


    void Foo() override
    {
        std::cout << "Derived" << std::endl;
    }    
};


int main()
{
    Base* base = new Base();
    Base* derived = new Derived();


    std::cout << sizeof(Base) << std::endl;
    std::cout << sizeof(Derived) << std::endl;    


    base->Foo();
    derived->Foo();


    delete base;
    delete derived;


    return 0;
}
```

Output of the PE:
```
8
8
Base
Derived
```

  With the introduction of virtual functions, the compiler generates a per-class structure called a vtable to support dynamic dispatch at runtime. As a result, object layout changes.

  Previously, an empty class had a size of 1 byte, which exists solely to ensure distinct object addresses. Once a class becomes polymorphic (i.e., has at least one virtual function), its size increases to the size of a pointer. On a 64-bit platform, this is 8 bytes.

  This extra storage is used to hold the vptr—a pointer to the class’s vtable.
For polymorphic objects, the first 8 bytes of the object layout contain the vptr.

Let’s examine the generated assembly for a virtual call:
```
# simple_virtual.cpp:39:     base->Foo();
	mov	rax, QWORD PTR -8[rbp]	 # tmp128, base
	mov	rax, QWORD PTR [rax]	 # _3, base_27->_vptr.Base
	add	rax, 16	 # _4,
	mov	rdx, QWORD PTR [rax]	 # _5, *_4
	mov	rax, QWORD PTR -8[rbp]	 # tmp129, base
	mov	rcx, rax	 #, tmp129
	call	rdx	 # _5
 # simple_virtual.cpp:40:     derived->Foo();
	mov	rax, QWORD PTR -16[rbp]	 # tmp130, derived
	mov	rax, QWORD PTR [rax]	 # _6, derived_36->_vptr.Base
	add	rax, 16	 # _7,
	mov	rdx, QWORD PTR [rax]	 # _8, *_7
	mov	rax, QWORD PTR -16[rbp]	 # tmp131, derived
	mov	rcx, rax	 #, tmp131
	call	rdx	 # _8
```

That’s a whole lot more instructions!
Let’s dissect them one by one with the use of *GDB*.

`mov rax, QWORD PTR -8[rbp]`
This loads the address of the object (base) from the stack.

`mov rax, QWORD PTR [rax]`
  Since the vptr is stored at offset 0 in the object, dereferencing the object pointer yields the vtable address.

**GDB:**
```
(gdb) x/4gx *(void**) base
0x7ff67eb745a0 <_ZTV4Base+16>:  0x00007ff67eb72980      0x00007ff67eb72950
0x7ff67eb745b0 <_ZTV4Base+32>:  0x00007ff67eb728d0      0x0000000000000000

(gdb) i symbol 0x00007ff67eb72980
Base::~Base() in section .text 
(gdb) i symbol 0x00007ff67eb72950
Base::~Base() in section .text of
(gdb) i symbol 0x00007ff67eb728d0
Base::Foo() in section .text 
```

  When the vptr is dereferenced, we do not land at the start of the vtable object (==_ZTV4Base==).
Instead, the vptr already points 16 bytes into the vtable, at the first virtual function entry following Itanium C++ ABI convention. That 16 bytes *hold offset to top* value and *RTTI* ptr. You can have a look here:

**GDB:**
```
(gdb) x/6gx *(void**) base - 16
0x7ff67eb74590 <_ZTV4Base>:     0x0000000000000000      0x00007ff67eb74540

(gdb) i symbol 0x00007ff67eb74540
__fu1__ZTVN10__cxxabiv117__class_type_infoE in section .rdata 
```

`*(void**) base` is equivalent to dereferencing the vptr.

*rax* now holds the address **0x7ff67eb745a0**.

`add	rax, 16`
  
  Since the function ptr offset in the table is already known at compile time, the instruction simply offsets the address 16 bytes to **0x7ff67eb745b0**. Things will be more complicated when it comes to multiple inheritance with the use of offset to top field.

`mov	rdx, QWORD PTR [rax]`

The function pointer is now loaded inside the *rdx* register.
```
	mov	rax, QWORD PTR -16[rbp]	 # tmp131, derived
	mov	rcx, rax	 #, tmp131
```
The next 2 move instructions are the same as non-virtual ones. They prepare an object pointer (*this*) as the first argument for next call instruction.

`call	rdx`

And finally, the function is invoked via an indirect call through an address stored inside *rdx*, resolving the correct override. 

Quite a lengthy one, right? With the presence of virtual functions, another level of indirection is introduced and can only be resolved at runtime. 

These instructions apply the same to *derived->Foo()*
Instead of going back over again, let’s examine its vtable:

**GDB:**
```
0x7ff67eb745c0 <_ZTV7Derived>:  0x0000000000000000      0x00007ff67eb74550
0x7ff67eb745d0 <_ZTV7Derived+16>:       0x00007ff67eb72a60      0x00007ff67eb72a30
0x7ff67eb745e0 <_ZTV7Derived+32>:       0x00007ff67eb729c0      0x0000000000000000

(gdb) i symbol 0x00007ff67eb74550
__fu0__ZTVN10__cxxabiv120__si_class_type_infoE in section .rdata
(gdb) i symbol 0x00007ff67eb72a60
Derived::~Derived() in section .text  
(gdb) i symbol 0x00007ff67eb72a30
Derived::~Derived() in section .text 
(gdb) i symbol 0x00007ff67eb729c0
Derived::Foo() in section .text 
```

In the next blog, we will investigate about how multiple inheritance works under the hood, how vtable, vptr interact with each other when the relation between child and parents gets more complex.
