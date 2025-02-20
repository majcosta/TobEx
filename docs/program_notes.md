## Program Notes

Things to look out for when programming for TobEx

- [Heap space](#heap-space)
- [Detouring procedures](#detouring-procedures)
- [Getting return addresses](#getting-return-addresses)

### Heap space
`TobEx.dll` and `BGMain.exe` use different executable heap spaces. This means that you cannot allocate memory to objects in one heap space and free it using another heap space, and vice versa.

`TobEx.dll` utilises `IENew` to allocate memory using `BGMain.exe` heap space, which overrides the global operator `new(size_t, int)` procedure.

While `TobEx.dll` allows freeing memory from `BGMain.exe` heap space using operator `delete(void*, int)`, where `int` is any value, this does not implicitly call the deconstructor for the object being freed. If the object contains pointers to other objects, their memory will not be freed, causing memory leaks.

The only way around this is to implement object-specific operator `new(size_t)` and operator `delete(void*)`, of which their procedure bodies will call `::operator new(size_t, int)` and `::operator delete(void*, int)`, respectively. This allows the simple use of new and delete to allocate and free memory, but restricts virtual memory use to the `BGMain.exe` heap space.

As a general rule, local variables in a procedure should use the local heap space. If virtual memory is required for any object in the game, always use the heap space of `BGMain.exe` to allocate and free memory.

### Detouring procedures
While detouring global procedures are straightforward, object-specific procedures are a little more tricky to detour. This is best illustrated in an example.

Let's say we wish to detour `int Stats::CalculateBonus(int a, int b)`.

We first define the object and include the procedure definition:

```c++
class Stats { //Size 8h
public:
  int CalculateBonus(int a, int b);

  int nMultiplier;
  int nAdder;
}
```

Then we define a function pointer to our desired procedure. Prepend the name of the function pointer with the object name to make it unique.

```c++
extern int (Stats::*Stats_CalculateBonus)(int, int); //in header file

int (Stats::*Stats_CalculateBonus)(int, int) =
  SetFP(static_cast<int (Stats::*)(int, int)>	(&Stats::CalculateBonus),	0x549F34);
```

This method of defining the function pointer can only be applied successfully to Visual C++ implementations of objects of none or single inheritance because the `SetFP` procedure will assume that the size of the function pointer is 4 bytes. Other compilers have different implementations of function pointers. Do not try to define a function pointer to procedures where the object has multiple inheritance.

Before getting onto the detour object, define the original procedure to call the function pointer:

```c++
int Stats::CalculateBonus(int a, int b) {
  return (this->*Stats_CalculateBonus)(a, b); }
}
```

We then define a detour object that will be a child to our parent object with the target procedure and define a related detour procedure:

```c++
class DETOUR_Stats : public Stats {
public:
  int DETOUR_CalculateBonus(int a, int b);
}
```

Since Microsoft Detours detours procedures with trampolines, if we use the original function pointer as the detour, other sections of TobEx code that call the function will jump to the trampoline instead of the normal function. This is undesired behaviour. Therefore, we will make a second function pointer that is a copy of the original function pointer. Prepend this with `Tramp_`.

```c++
extern int (Stats::*Tramp_Stats_CalculateBonus)(int, int); //in header file

int (Stats::*Tramp_Stats_CalculateBonus)(int, int) =
  SetFP(static_cast<int (Stats::*)(int, int)>	(&Stats::CalculateBonus),	0x549F34);
```

What do we have here? If we call `(this->*Stats_CalculateBonus)(a, b)`, we will jump to address `0x549F34`. If we call `(this->*Tramp_Stats_CalculateBonus)(a, b)`, we will jump to the trampoline that the detour has set up. Therefore, we use the former when other unrelated TobEx code wants to use `CalculateBonus()`, whereas we use the latter if we want our `DETOUR_CalculateBonus() to call back CalculateBonus()`. So in the simplest case, we define our detour function:

```c++
int DETOUR_Stats::DETOUR_CalculateBonus(int a, int b) {
  return (this->*Tramp_Stats_CalculateBonus)(a, b);
}
```

If we used `(this->*Stats_CalculateBonus)(a, b)` instead, we will crash the program with a stack overflow.

To finally initiate the detour, all we need is to use `DetourMemberFunction()`:

```c++
DetourMemberFunction(Tramp_Stats_CalculateBonus, DETOUR_Stats::DETOUR_CalculateBonus);
```

This sets up and commits the detour transaction for us.

### Getting return addresses

TobEx implements a simple method of getting the return address to identify the calling function. You use `GetEip(DWORD)`. This utilises the function header via assembly code to grab the return address to the calling function. Always put this at the beginning of your function, else `GetEip()` will grab useless data.

Let's say we want the calling function to our `CalculateBonus()` function above. All we need is:

```c++
int DETOUR_Stats::DETOUR_CalculateBonus(int a, int b) {
  DWORD Eip;
  GetEip(Eip);

  return (this->*Tramp_Stats_CalculateBonus)(a, b);
}
```
