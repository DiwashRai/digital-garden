---
title: Effective C++
tags:
  - source
  - textbook
enableToc: true
openTOC: true
---

Author: [[Scott Meyers]]  
Topics: [[Software Engineering]]  

---

## Ch1: Accustoming yourself to C++  

### **Item 1:** View C++ as a federation of languages  
C++ is better understood as a group of related sub-languages. These are:
- C
- Object-oriented C++
- Template C++
- The STL

This is important to keep in mind as different sub-languages have different effective strategies and different conventions.


### **Item 2:** Prefer consts, enums, and inlines to \#defines  
Rule could also be called "Prefer the compiler over the preprocessor".   

```cpp
// BAD
#define ASPECT_RATIO 1.653

// GOOD
const double AspectRation = 1.653;
const char* const authorName = "Scott Meyers";
const std::string authorName("Scott Meyers");
```

The preprocessor will blindly substitute ASPECT_RATIO for 1.653 resulting in multiple copies in the object code.   

```cpp
// BAD
#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a) : (b))

// GOOD
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a: b);
}
```
Function like macros can be very error prone. Calling `CALL_WITH_MAX(++a, b)` for example results in a being incremented twice.


### **Item 3:** Use const whenever possible  
- Helps detect usage errors.
- Using `const` in function declaration is extremely powerful so make use of it. It can refer to the function's return value, the individual parameters or even the function as a whole.
- Compilers enforce 'bitwise constness' but you should program using 'logical constness'.
	- Bitwise constness/Physical: does not modify any of object's data members
	- Logical constness: Some data members of an object can be modified but should be in ways the client cannot detect. Achieved using `mutable`. Also considers situations where something is bitwise const, but does not behave const. e.g. the altering of data members by returning non-const references.
- When const and non-const member functions have similar implementations, avoid duplication by calling const version in the non-const function and using `const_cast`.


### **Item 4:** Make sure objects are initialised before use
The rules about when objects initialisation is guaranteed or not is too complicated to be worth memorising. Avoid the issue by ==always initialising==. 
- For non-member built in types, this needs to be done manually. e.g. `int x = 0;`
- For the rest, initialisation is the responsibility of the constructors and the use of initialisation list is preferred.


## Ch2: Constructors, destructors and assignment operators  

### **Item 5:** Know what functions C++ silently writes and calls
Compilers will declare the following if you do not do so yourself:
- Copy constructor
- Copy assignment operator
- Destructor

If no constructors are declared at all, compilers will also declare:
- Default constructor

### **Item 6:** Explicitly disallow the use of compiler-generated functions you do not want
An example of an unwanted generated function is a copy constructor and copy assignment operator for objects that should not be copied.  

```cpp
HomeForSale h1;
HomeForSale h2;
HomeForSale h3(h1); //should not compile
h1 = h2;            // should not compile
```

To achieve this there is a ==well known trick==. Declare the functions private and do not implement them.

```cpp
class HomeForSale {
public:
    ...
private:
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(ocnst HomeForSale&);
}
```

This trick works and generates a link-time error. It is possible to move
it up to a compile time error by declaring the functions private in a base
class designed just to prevent copying.

>[!abstract] Summary
>- Disallow compiler generated function by declaring the function private and not providing an implementation

### **Item 7:** Declare destructors virtual in polymorphic base classes
Lets say you have a base class `TimeKeeper` and some derivided classes
that inherit from it:  
- AtomicClock
- WaterClock
- WristWatch

If you try to delete a derived class (e.g. WristWatch) with a base class
pointer (TimeKeeper), the results are undefined. Most likely, the derived
class parts of the object won't be deleted. This is a good way to leak
memory.

Solution is simple: ==give the base class a virtual destructor==.

However, when a class is not meant to be used as a base class, making it
virtual is usually a bad idea. This is because virtual functions require the
use of vptr ("virtual table pointers") which increases the size of the type.
A simple data type that contains two ints, would go from 64bits in size to
128bits. An increase of 100%. It also makes the object less portable to C.

If you want to make a class abstract that does not have functions to make
virtual, you can just give it a pure virtual destructor. This is because a
abstract classes are meant to be base classes and base classes should
have a virtual destructor.

> [!abstract] Summary
> - Polymorphic base classes should declare virtual destructors.
> - Any class with virtual functions should have a virtual destructor.
> - Classes that aren't base classes should not declare virtual destructors.
> - Use a pure virtual destructor to make classes abstract that do not have functions

### **Item 8:** Prevent exceptions from leaving destructors  
C++ allows destructors to emit exceptions but it is heavily discouraged.
This is because if an exception is thrown, for example, in a vector
containing 10 widgets, the rest of the widgets still need to be deleted or
there will be leak. However, if you then have another exception when
deleting the other widgets, now you have two simultaneous exceptions
which results in undefined behaviour.

Example class that closes database connection in destructor:
```cpp
class DBConn {
public:
    ...
    ~DBConn()
    {
        db.close();
    }
private:
    DBConnection db;
}
```
This will cause issues if the call to close results in an exception. The
exception will be propagated.

There are two options to fix this:  
- ==Terminate the program== (e.g. with std::abort()). This is a reasonable solution if the program can no longer run due to the error.
- ==Swallow the exception==. This is a bad idea as we need to know when something fails.

The actual ideal solution is to create a function that closes the connection
in the DBConn interface so the client can react to the exception. Then
the backup call to 'close()' can still be placed in the destructor. This may
seem like it goes against *item 11*, by placing an extra burden on the client,
however it is not as it actually **gives** them a chance to react to the
exception.

> [!abstract] Summary
> - Destructors should never emit exceptions.
> - If a classes clients need to react to exceptions during an operation, offer a regular 'non-destrcutor' function that performs the operation.

### **Item 9:** Never call virtual functions during construction or destruction
Don't call virtual functions during construction or destruction as they won't
do what you expect them to. If they did, it would lead to undefined
behaviour.

Example: Say you have a base class `Transaction` that is inherited by a 
class called `BuyTransaction`. If you have a virtual function in Transaction
that is called during construction, it will call the base class version. Not
the derived version. This is because the base class is constructed first so
the derived members will not have been constructed yet.

> [!abstract] Summary
> - Don't call virtual functions from constructors or destructors.

### **Item 10:** Have assignment operators return a reference to `this`
Assignments can be chained together and is right associative.
```cpp
int x, y, z;
x = y = z = 15;
//equivalent to
x = (y = (z = 15));
```
This is implemented by assignments returning a reference to it's left hand argument. This is the convention that should be followed.

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        ...
        return *this;
    }
    ...
}
```

> [!abstract] Summary
> - Have assignment operators return a reference to `*this`

### **Item 11:** Handle assignment to self in operator=.
This looks silly but is allowed:
```cpp
Widget w;
...
w = w;
```
``
If it's allowed, clients will end up doing it. A less obvious version may look
like this:
```cpp
// potential assignment to selfs
a[i] = a[j];
*px = *py;
```

When writing resource managing classes you can fall into the trap of
accidentally releasing a resource before you're done with it:
```cpp
class Bitmap {...};
class Widget{
    ...
private:
    Bitmap *pb;
}

Widget& Widget::operator=(const Widget& rhs)
{
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

Here you delete rhs not realising that it is the same as `this`. The
traditional fix is this:
```cpp

Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this;
    
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
This is now self-assignment safe but not exception safe. If `new Bitmap`
causes an exception, it will still cause issues.

Making operator= exception-safe also typically also renders it
self-assignment-safe. It is common to deal with self-assignment issues by
ignoring them and isntead focusing on exception safety. A careful ordering
of statements can make code exception safe. For example:

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap *pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;

    return *this;
}
```

> [!tip] Performance Hint  
> The identity test can still be placed at the top for efficiency, however,
> consider how often self-assignment occurs. The check is not free. It
> makes the source code and object bigger and introduces a branch.
>

Alternative to manual ordering is the "copy and swap" technique.
```cpp
class Widget {
    ...
    void swap(Widget& rhs); // exchanges *this and rhs's data
    ...
};

Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);
    swap(temp);
    return *this;
}
```

The final variation sacrifices some clarity for allowing the compiler to
sometimes generate more efficient code.
```cpp
Widget& Widget::operator=(Widget rhs) // rhs is a copy of the object
{
    swap(rhs);

    return *this;
}
```

> [!abstract] Summary  
> - Make sure operator= can handle self-assignment in a well-behaved
> manner
> - Make sure any function operating on more than one object behaves
> well if two or more of the objects are the same.

### **Item 12:** Copy all parts of an object.
Well-designed objects contain only two functions that copy objects: the
copy constructor and copy assignment operator. Compilers will generate
these functions if required and do what you expect them to: copy all the
data of the object being copied.

However, when you declare your own copy functions, they do not warn you
when your implementation is wrong. You may forget to copy a newly
added data member and there will be no warnings.

Another way that you may run into issues is when using inheritance.
```cpp
class PriorityCustomer:public Customer {
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```

This may look fine and appear to be copying everything, however, 
crucially, they are not copying the data members it inherits from customer.
The fix to this would be the following:
```cpp

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: Customer(rhs)                 // invoke base class copy ctor
  priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs);  // assign base class parts
    priority = rhs.priority;
    return *this;
}
```

> [!warning] Warning  
> The two copy functions will have similar code but do not call one from
> the other as it is a non-sensical operation. If you really must, have the
> two call a third private function such as init().

> [!summary] Summary
> - Make sure copy functions copy all of an object's data members
> including the base class parts.
> - Don't implement one copy function in terms of the other. Put common
> code in a third function.

## Ch3: Resource management  

### **Item 13:** Use objects to manage resources  
Managing resources manually is very error prone and should be avoided.
Consider the following:
```cpp
void f()
{
    Investment * pInv = createInvestment(); // factory function
    ...                                     // use pInv

    delete pInv;                            // release object
}
```
This may look reasonable but there are many ways it could fail to delete
`pInv`. if there is a pre-mature return within the function, control would
never reach the delete. Another situation is if `createInvestment()` and
`delete pInv` were in a loop that then breaks early. When this happens,
there will be a leak which will also affect any other resources helf by
*that* object.

Technically, carefull programming an avoid this issues, but this is not
realistic. As the code is maintained, someone may add a pre-mature return
without realising the consequences. Or someone may add an exception in a
function that did not use exceptions which will then cause a leak.

To avoid all of this, we can put resources into a resource managing object
which will then free the resources automatically when the destructor is
called.

The STL has smart pointers that are tailormade for resources that are
dynamically allocated on the heap and are used only within a single
block or function.

```cpp
void f()
{
    std::unique_ptr<Investment> pInv(createInvestment());
    ...
}
```
This example demonstrates two critical aspects of using objects to manage
resources:
- ==**Resources are acquired and immediately turned over to
resource-managing objects.**==  The idea that resources are acquired and
immediately turned over to a resource managing object is referred to as
*Resource Acquisition is Initialization*(RAII).
- ==**Resource-managing objects use their destructors to ensure that
resources are released.**== Destructors are automatically called when an object
goes out of scope so resources are released regardless of how control
leaves a block.

> [!abstract] Summary  
> - To prevent leaks, use RAII objects that acquire objects in their
>constructor and delete them in their destructor
>- Two most common RAII classes are std::unique_ptr and
>std::shared_ptr

### **Item 14:** Think carefully about copying behaviour in resource-managing classes  
Not all resources are heap based. For these resources, unique_ptr and
shared_ptr are generally inappropriate as resource handlers.

For example, if you create a RAII  class called `Lock` that unlocks a mutex
in it's destructor so you never forget to unlock a mutex. What should
happen if this object is copied?
```cpp
Lock ml1(&m);
Lock ml2(ml1);
```

There are a few valid options:
- **Prohibit copying:** In many cases it makes no sense to copy a RAII object.
In these situations you should just prohibit it (item 6). This is likely to be
true for an object such as `Lock`.
- **Reference-count the underlying resource:** Sometimes you may want to
hold on to a resource until there are no more objects using it. In this
situation a copy means that the count of objects referring to the resource
is incremented. A shared_ptr should be used to implement this behaviour.
shared_ptr also allows using a custom "deleter" where you can specify
a different behaviour (such as unlocking a mutex) instead when the
reference count goes to 0.
- **Copy the underlying resource:** Sometimes you can have as many copies
as you want of a resource. The only reason to have a resource-managing
class would be to ensure resources are released. In this case, copying the
RAII object should also mean copying the resource it wraps. An example
would be std::string.
- **Transfer ownership of the underlying resource:** In rare situations, you
may want to ensure that only one RAII object refers to a resource. Here
you will want to make sure that a copy means that ownership is
transferred.

Remember that copying function will be generated by the compiler so you
will need to need to write the custom behaviour yourself.

> [!abstract] Summary  
> - Copying an RAII object entails copying the resource it manages. The
> correct behaviour of the RAII object depends on the copying behaviour
> of the underlying resource.
> - Common RAII class copying behaviours are: disallowing copy and
> reference counting. However, there are others.

### **Item 15:** Provide access to raw resources in resource-managing classes  
Resource-managing classes are great but they can't always be fully relied
on. Many APIs refer to the resources directly so from time to time you will
need to deal with raw resources.
```cpp
int daysHeld(const Investment* pi);
```

There are two ways to convert a RAII object to the resource it contains:
explicit conversion and implicit conversion.

Smart pointers contain a `get` member function that performs an explicit
conversion i.e. returns a copy of the raw pointer inside the smart pointer
object.
```cpp
int days = daysHeld(pInv.get());
```

As for implicit conversion, it is a bit more dangerous but can sometimes
still be the correct choice. Here is an example of a Font class that
manages an underlying FontHandle resource with an implicit conversion
function.
```cpp
class Font {
public:
    ...
    operator FontHandle() const { return f; }
    ...
private:
    FontHandle f;
};

// explicit conversion used to call changeFontSize
changeFontSize(f.get(), newFontSize);
// implicit conversion version of same code
changeFontSize(f, newFontSize);
```

However, now a client might accidentally create a FontHandle when a font
was intended.
```cpp
Font f1(getFont());
...
FontHandle f2 = f1;
// meant to copy a font object but implicityly converted f1
// into its underlying FontHandle and then copied that
```
Now the same FontHandle is being used in f1 and f2 which will then result
in a dangling pointer.

> [!abstract] Summary  
> - APIs often require access to raw resources so RAII classes should
> offer a way to access the resource it manages.
> - Access may be explicit or implicit. Explicit is safer but implicit is
> more convenient.

### **Item 16:** Use the same form in corresponding uses of `new` and `delete`  
```cpp
std::string *stringArray = new std::string[100];
...
delete stringArray;
```
The above code is wrong as you are constructing an array of objects but
deleting only 1 object. `new` allocates only enough space for an object and
then constructs that object if its a single object. For an array, it also
includes the memory required to store information on the size of the array.
Therefore, when you delete an array, you need to access the information.
This can only be done if you tell the compiler that the information is there
by using `delete []`.

> [!warning] Warning  
> - Make sure to use same for of `new` in all of your constructors so you
> know which form to call in the destructor.
> - Abstain from using array types in `typedefs` so there is no confusion
> on which form of `delete` to call when using a typedef.

> [!abstract] Summary  
> - Match the forms of `new` and `delete`. `new` requires a corresponding
> `delete` and `new []` would require a corresponding `delete []`.

### **Item 17:** Store newed objects in smart pointers in standalone statements  
Consider a situation where we have a function that returns a priority and
a function that processes a dynamically allocated Widget.
```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```

Now consider the following way to call processWidget.
```cpp
processWidget(std::shared_Ptr<Widget>(new Widget), priority());
```

Three things need to happen:
- Call priority()
- Execute 'new Widget'
- Call the shared_ptr constructor

Compilers have leeway over which order these things are done. The order
they are done could be:
- Execute 'new Widget'
- Call priority()
- Call shared_ptr constructor

The problem can now be seen. If there is an exception when priority() is
called, the pointer returned from 'new Widget' will be lost and not stored
in the shared_ptr we used to guard against leaks.

To prevent this, use a separate statement to create and store the Widget
resource.
```cpp
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```
The revised code has less leeway to reorder the various components that
made the previous 1 line version of these two statements.


> [!summary] Summary
> - Store newed objects in smart pointers in standalone statements.
> Not doing so can lead to subtle resource leaks if exceptions are thrown.


## Ch4: Designs and declarations  

### **Item 18:** Make interfaces easy to use correctly and hard to use incorrectly.  
If clients use your code incorrectly, ==your interface is partially to blame==.
Consider the design below for a Date class:
```cpp
class Date {
public:
    Date (int month, int day, int year);
    ...
};
```
Two errors can easily happen:
- Client passes in day value into month. `Date d(30, 3, 1995);`
- Client passes in invalid value for month or date. `Date d(3, 40, 1995);`

Many similar client errors can be fixed by using simple wrapper with
explicit constructors.
```cpp
struct Day {
    explicit Day(int d)
    : val(d) {}

    int val;
};
Date::Date(const Month& m, const Day& d, const Year& y);
```

Full fledged classes for day, month and year would be even better but
even structs are enough to show that introducing types can prevent a
interface usage errors.

Once the right types are being used, you can then progress to limiting the
values of those types if it is reasonable to do so. Enums can be used to do
this but they are not as typesafe as we would like as they can be used like
ints.

```cpp
class Month {
    static Month Jan() { return Month(1); }
    static Month Feb() { return Month(2); }
    ...
}
Date d(Month::Mar(), Day(30), Year(1995));
```

> [!tip] Hint  
> This item was probably written before scoped enums (enum classes)
> were available. Now you probably could replace this with enum classes.

*Another way to prevent client errors is to add const.* For example you
can const qualify operator* which can prevent this error  
`if (a * b = c)...`  
This is an assignment when it should have been a comparison.

*Try to make your interfaces behave consistently as possible.* Consistency
is one of the most important characteristic to making easy to use
interfaces. Conversely, inconsistency leads to aggravating interfaces

*Any interface that relies on the client to remember to do something is
prone to incorrect use, because clients cant forget to do it.* An example
is returning raw pointers.  
`Investment* createInvestment();`  
This function creates the opportunity for two errors: not deleting the
pointer and deleting it twice. This can be avoided by returning a smart
pointer in the first place.  
`std::shared_ptr<Investment> createInvestment();`  
In fact shared_pt also make sit possible to get rid of other errors with the
ability to specify a custom release function (deleter). It can also get rid
of the cross-DLL problem. This is a problem where an object is created
using `new` in one DLL but is deleted in a different DLL. shared_ptr gets
rid of this as its default deleter uses `delete` from the same DLL where
shared_ptr is created.

> [!abstract] Summary  
> - Strive to create interfaces that are easy to use correctly but difficult
> to use incorrectly.
> - For correct use: maintain consistency in interfaces and use built-in
> types for behavioural compatiblity.
> - For error prevention: create new types, restrict operations on types,
> constrain object values, and eliminate client resource management
> responsibilities.
> - std::shared_ptr supports custom deleters. This prevents cross DLL
> problem and can be used to automatically unlock mutexes.

### **Item 19:** Treat class design as type design.  
Look at class designing as type designing. This means as a C++ developer
you are a *type designer*. Designing good classes is a challenge because
designing good types are a challenge.

To understand the issues you face which will help you design your type
you should consider the following questions:
- How should objects of your new type be created and destroyed?
- How should object initialisation differ from object assignment?
- What does it mean for objects of your new type to be passed by value?
- What are the restrictions on legal values of your new type?
- Does your new type fit into an inheritance graph?
- What kind of type conversion are allowed for your new type? - consider
implicit and explicit conversions. You may need conversion functions if you
are looking to make it explicit only.
- What operators and functions make sense for the new type?
- What standard functions should be disallowed? - declare them private or
delete them(C++11).
- Who should have access to the members of your new type?
- What is the "undeclared interface" of your new type? - what guarantees
does it offer with respect to performance, exception safety and resource
usage.
- How general is your new type?
- Is a new type really what you need? - if it's a family of types, you
might want a class template.

> [!abstract] Summary  
> Class design is type design. When designing a new type, consider all 12
> questions listed in this item.


### **Item 20:** Prefer pass-by-reference-to-const to pass-by-value.  
C++ passes by value by default - function parameters are initialised with
copies of the actual argument and function callers get back a copy of the
returned value. These copies are produced by the objects copy constructor
which makes it an expensive operation.

Consider the following:  
```cpp
class Person {
public:
    Person();
    virtual ~Person();
    ...
private:
    std::string name;
    std::string address;
};

class Student: public Person {
public:
    Student();
    virtual ~Student();
    ...
private:
    std::string schoolName;
    std::string schoolAddress;
};

// function declaration
bool validateStudent(Student s);

Student plato;
bool platoIsOk = validateStudent(plato);
```

When the validateStudent function is called the cost is 1 call to the
copy constructor to initialise the parameter and one call to the destructor
when the function returns. Furthermore, the strong objects are also copied
and destroyed. With construction and destruction of the 4 strings
accounted for, ther are 6 calls to constructors and destructors.

Pass by reference-to-const solves this. The const is essential as
previously the callers were shielded from any changes happening to the
Student object as it was a copy. Now that it is a reference, const is
required to provide that assurance.

Passig parameters by reference also prevents the *slicing problem*. This is
when you pass a derived class into a function that accepts the base class.
If you do this by value, the base class constructor will be called and the
derived class part of the object will not be copied. Then when you call the
function that may have been overriden in the derived class, the base class
version will be called.

If you pass by reference (or pointer) the object will behave like the actual
object that is passed in.

References are implemented as pointers under the hood of a C++ compiler.
This means that for built-in types (e.g.int) it is reasonable to choose to
pass-by-value.

Built-in types are small which can lead people to conclude that small types
can be good pass-by-value candidates. However, there are a few reasons
that this should not be done. Small types can sometimes be only small
as they contain only a pointer, however, the object that the pointer points
to might then be expensive to copy. Another reason is that compilers will
treat built-in types differently to user-defined types. For example,
compilers can refuse to put user-defined types that consist of just a
double in the register even if it will do so with a naked double. Finally,
user-defined types might change in the future so something that is small
now can increase in size later.

==Generally, only built-in types, STL iterators and function object types can
reasonably be assumed as good pass-by-value candidates==.

> [!abstract] Summary  
> - Prefer pass-by-reference-to-const over pass-by-value for efficiency
> and to sidestep the slicing problem.
> - The exceptions to the rule are built-in types, STL iterators and
> function object types.

### **Item 21:** Don't try to return a reference when you must return an object.  
Do not be over zealous in the pursuit of eliminating the 'evil' of
pass-by-value. This can lead to passing references of objects that do not
exist.

Consider a class representing rational numbers.
```cpp
class Rational {
public:
    Rational(int numerator = 0,
             int denominator = 1);
    ...
private:
    int n, d;
friend:
    const Rational
      operator*(const Rational& lhs,
                const Rational& rhs);
};
```

The `operator*` function returns the result object by value. The question is,
can you avoid the cost of the object's deconstruction and construction?

Example scenario:
```cpp
Rational a(1,2); // 1/2
Rational b(3,5); // 3/5

Rational c = a * b; // 1/2 x 3/5 = 3/10
```

It may be tempting to consider the following implementation:
```cpp
const Rational& operator*(const Rational& lhs,
                          const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result; // returns a reference
}
```
This is useless as the constructor and deconstructor is still being called.
More crucially though, the function returns a reference to a local object
which will immediately go out of scope and we are in undefined territory.

Now you might consider returning a reference to an object on the heap.
Yet again, this would be bad. You still have to pay for the constructor, but
now the client needs to remember to delete the newed object. And even if
they are, how do they handle a situation like this:  
```cpp
Rational w, x, y, z;
w = x * y * z;
```
There are two calls to operator* but now way to delete the first call. This
is a guaranteed resource leak.

Using a static 'result' object is also a no go. It will run into problems with
situations like this:
```cpp
Rational a, b, c, d;
if ((a * b) == (c * d))
    ...
```
This will always be true as there are two active calls to operator* when
operator== is called. They will always be equal as they share the static
result object.

The right answer is simply something such as:
```cpp
inline const Rational operator*(const Rational& lhs,
                                const Rational& rhs)
{
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

> [!abstract] Summary  
> Never return a pointer or reference to a local stack object, a reference
> to a heap-allocated object, or a pointer or reference to a local static
> object if there is a chance that more than one such object will be
> needed.

### **Item 22:** Declare data members private.  
The case for this item will follow this format: highlight why public data
members is bad -> highlight why all the arguments also apply to protected
members -> conclude that private is the way to go.

If everything in the public interface is a function, clients won't have to
recall whether to use parantheses to access members or not. Consistency.

Functions provide fine-grained control over the accessiblity of it's data
members.

Finally, encapsulation. If access to a data member is via a function, it is
possible to then later replace the data member with a computation and your
clients will not be aware or need to be aware at all.

A quick example is a class that contains speed data:
```cpp
class SpeedDataCollection {
    ...
public:
    void addValue(int speed);
    double averageSoFar() const;
    ...
}
```
You can use an approach that maintains a data member that contains the
running average and the `averageSoFar()` function just returns this value OR
an approach where the `averageSoFar()` is calculated from all the stored
values whenever it is called. One has better performance on
`averageSoFar()` at the cost of space and the other has worse performance
but doesn't take as much space.  
With encapsulation, you can interchange these easily.

==Encapsulation is more valuabled than it might initially appear==. If data
members are hidden, class invariants can be maintained. This has benefits
when it comes to threaded environments as well. ==Hiding details is like
reserving the right to change the implementation details at a later date==.
You will realise that even if you own the source code to a class, your
ability to change anything public will be severely limited.

*The argument for protected members is identical*. This is because
somethings encapsulation is inversely proportional to the amount of code
that could break if that something changes. For public members this can
be 'all client code' which is an unknowably large amount. How about for
protected members? The answer is 'all the derived classes that access it'
which is yet again an unknowably large amount of code.

==Essentially there are two levels of access: private and everything
else.==

> [!abstract] Summary  
> - Declare data members private.
> - Protected is no more encapsulated than private.

### **Item 23:** Prefer non-member non-friend functions to member functions.  
Consider a web browser class which may have functions to manage the
cache, history and cookies:
```cpp
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
};
```

Now consider two ways of implementing a function that combines all three.
```cpp
class WebBrowser {
    public:
    ...
    void clearEverything();
    ...
};

void clearBrowser(WebBrowser& wb) // non-member function
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

Which is better? The member or non-member function? A misunderstanding
of object oriented principles may lead people to conclude that data and
functions that operate on them should be bundled together. This is wrong.
Object oriented principles dictate that data should be as *encapsulated* as
possible.

The greater something is encapsulated, the greater our ability to change
it. Adding another member function increases the public functions that
access the private data members. In the example it would go from 3 to 4.
However, the non-member function does not affect the number of
functions that can access the private parts of the class. Therefore, it is
more encapsulated with a non-member function.

A way to 'bundle' convenience/utility functions without affecting
encapsulation is to use namespaces. This is a very flexible approach as
convenience functions can be spread out over different files but in the
same namespace. Clients can then selectively `#include` only the parts
that they require. It also allows for easy extensibility by the clients
or by the writers in future.

> [!abstract] Summary  
> Prefer non-member non-friend fucntions to member functions. Doing so
> increases encapsulation, packaging flexibility, and functional
> extensibility.

### **Item 24:** Declare non-member functions when type conversions should apply to all parameters.  
Classes supporting implicit type conversions is generally a bad idea,
however there are exceptions. A common exception is for numerical types.

Consider a `Rational` class:
```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
private:
    ...
};
```

Next lets say you want to support multiplication of Rational numbers. The
question is should it be implemented as a member function, non-member
function or non-member function that are friends.

Well lets consider the member function version:
```cpp
class Rational {
public:
    ...
    const Rational operator*(const Rational& rhs) const;
};
```

This design allows for the following:
```cpp
Rational oneEighth(1,8);
Rational oneHalf(1,2);

Rational result = oneHalf *oneEighth;  //fine
result = result * oneEighth;           //fine
```

However, you also want to support mixed-mode operations where Rationals
might be multiplied with ints for example. Lets see what happens.

```cpp
result = oneHalf * 2;       //fine
result = 2 * oneHalf;       //error
```

The source of the problem is clearer if you rewrite it:
```cpp
result = oneHalf.operator*(2);    //fine
result = 2.operator*(oneHalf);    //error
```

oneHalf is an instance of a class containing the operator* where as 2 is
not. But then this raises the question, why does it work when you pass
in an int (2)? This is because of implicit conversion. Compilers can figure
out that although you are passing an int into a function that requires a
`Rational`, the int can be use to construct the `Rational` so that's what it
does.

This means that if the constructor for `Rational` was declared explicit,
then it would fail.

Anyhow, the key to the solution of supporting operations both ways around
is that implicit conversion is only available for parameters that are listed
in the parameter list. Here is the solution:

```cpp
class Rational {
    ...   // operator* not defined/declared
};

const Rational operator*(const Rational& lhs,
                         const Rational& lhs);
{
    ...
}

Rational oneFourth(1, 4);
Rational result;

result = oneFourth * 2;
result = 2 * oneFourth;   // both finally working
```

Should we make this function a friend function? No. It can already achieve
what it needs to do without breaking encapsulation. A function should not
be made a friend just because it is 'related' to it.

> [!abstract] Summary  
> If you need type conversions on all parameters to a function including
> the one pointed to by `this`, the function must be a non-member.

### **Item 25:** Consider support for a non-throwing swap.  
Swap is an interesting function that is worth some special consideration.
Here is a default implementation for swap:

```cpp
namespace std {
    template<typename T>
    void swap(T& a, T& b)
    {
        T temp(a);
        a = b;
        b = temp;
    }
}
```

This default implementation may not thrill you as it involves copying three
objects. For some types, none of this copies will be necessary. For types
like this, the default swap implementation will slow down your code.

The most common example for types like these are those consisting
primarily of pointers to another type that then contains the real data. A
common manifestation of this design approach is the "pimpl idiom"
("pointer to implementation" - item 31). An example using a Widget class:

```cpp
class WidgetImpl {
public:
    ...
private:
    int a, b,c;             //possibly lots of data
    std::vector<double> v;  //expensive to copy
};

class Widget {
public:
    Widget(const Widget& rhs);
    Widget& operator=(ocnst Widget& rhs)
    {
        ...
        *pImpl = *(rhs.pImpl);
        ...
    }
    ...
private:
    WidgetImpl *pImpl;
};
```

In this example, all we need to do to swap Widget objects is to swap
their underlying `pImpl` pointers. However, the default swap function does
not know this. It would result in copy three widgets and ALSO three
WidgetImpl objects. Very inefficient.

We would like to tell std::swap how to more efficiently handle the swap.
```cpp
namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        swap(a.pImpl, b.pImpl);  // wont compile. private ptrs
    }
}
```
The "template<>" at the begining of the fucntion says that this is a *total
template specialisation* for std::swap and the "< Widget >" after the name
of the function says that the specialisation is for when T is Widget.

In general, we are not permitted to alter contents of the std namespace,
but we are allowed to totally specialise standard template for our own
types.

The above code won't compile due to `pImpl` being private. We could make
the specialisation a friend, but the convention is to declare a public
member function calle dswap that does the actual swapping, then
specialise the std::swap to call the member function.

```cpp
class Widget {
public:
    ...
    void swap(Widget& other)
    {
        using std::swap; //need for this explained below
        swap(pImpl, other.pImpl);
    }
    ...
};
namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b);
    }
}
```

This compiles AND is consistent with the STL containers.

But what if `Widget` and `WidgetImpl` were class *templates* instead of
classes. Putting a swap member function in Widget is as easy as before.
The trouble arises with the specialisation. We 'want' to write:
```cpp
namespace std {
    template<typename T>
    void swap<Widget<T> > (Widget<T>& a,
                           Widget<T>& b)
    {a.swap(b);}
}
```

This looks reasonable but is illegal code. C++ supports partial 
specialisation for classes but not function templates. To partially
specialise a function template, the usual approach is to simply use an
overload.
```cpp
namespace std {
    tempalte<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {a.swap(b);}
}
```

In general this would be fine, but the std namespace is special. You can
totally specialize templates in std, but you can't add new templates (or
classes or functions or anything else). If you break this rule, you are
once again in the land of `undefined`.

So what do we actually do? We still declare a non-member swap that calls
the member swap, we just don't declare the non-member to be a
specialisation or overloading of std::swap.

```cpp
namespace WidgetStuff {
    ...
    template<typename T>
    class Widget{...};
    ...

    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    { a.swap(b); }
}
```

Now, if any code anywhere calls swap on two Widget objects, the name 
lookup rules in C++ will find the Widget-specific version in WidgetStuff.

This approach works as well for classes as for class templates, so it 
seems like we should use it always. Unfortunately, there is a special
reason for specialising std::swap for classes, so if you want to have
your class specific swap be called in as many contexts as possible
(which you definitely do), you need to write a non-member version in
the same namespace as your class AND a specialisation of std::swap.

The above is for `swap` authors but we should consider one thing for th
clients.
```cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    ...
    swap(obj1, obj2);
    ...
}
```
Which swap does this call? The general one in std? or the specialisation of
the std? or a T-specific one? What you want to do is try to use the
T-specific one, but if not fall back on the general std::swap.  
Heres how you can do this:
```cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;        // make std::swap available
    ...
    swap(obj1, obj2);       // call best swap
    ...
}
```

The C++ lookup rules will ensure the best swap is called so don't worry
too much. Just remember not to qualify it:
`std::swap(obj1, obj2);`  
Qualifying swap does happen, that's why it's in your best interest to aslo
implement the specialised version.

> [!abstract] Summary  
> - Provide a swap member function when std::swap would be 
> infefficient for your type. Make sure swap does not throw exceptions.
> - If you offer a member swap, make sure to also offier a non-member
> swap that calls the member. For classes (not templates), specialise
> std::swap too.
> - When calling swap, employ a using declaration for std::swap, then
> call swap without namespace qualification.
> - It's fine to totally specialise std templates for user-defined type,
> but never try to add something completely new to std.

## Ch5: Implementations  
The lion's share of the battle is coming up with appropriate definitions for
your classes and appropriate declarations for your functions. However,
there are still implementation details to look out for.

### **Item 26:** Postpone variable definitions as long as possible.  
Defining a variable of a type with a constructor or destructor incurs the
cost of construction and then the cost of destruction when the variable
goes out of scope. Due to the inherent cost of using a variable, you want
to avoid using them if possible.

Consider a function that returns an encrypted version of a password if the
password is long enough:
```cpp
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    string encrypted;

    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...      // encrypted password into encrypted var
    return encrypted;
}
```

The object encrypted is unused an exception is thrown. Therefore, it's
better if you postpone defining it until you know an exception won't be
thrown.
```cpp
std::string encryptPassword(const std::string& password)
{
    using namespace std;

    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    string encrypted;
    ...      // encrypted password into encrypted var
    return encrypted;
}
```

However, even this version isn't tight enough as `encrypted` is defined
without any initialisation arguments. This means it is default constructed
and then only assigned a value later.

Consider if the hard part of the function was done in this function:  
`void encrypt(std::string& s);`  

The code could then look like this:  
```cpp
std::string encryptPassword(const std::string& password)
{
    ...
    string encrypted;
    encrypted = password;

    encrypt(encrypted);
    return encrypted;
}
```

This leads to a pointless default construction of `encrypted` though. Better
would be:  
```cpp
std::string encryptPassword(const std::string& password)
{
    ...
    string encrypted(password);
    encrypt(encrypted);
    return encrypted;
}
```

==Not only should you postpone variable definition until right before you
have to use the variable, but also until you have the initialisation
parameters for it.==

But what about loops?
```cpp
// Approach A: define outside loop
Widget w;
for (int i = 0; i < n; ++i) {
    w = some value dependent on i;
    ...
}

// Approach B: define inside loop
for (int i = 0; i < n; ++i) {
    Widget w(some value dependent on i);
}
```

The associated costs are:
- Approach A: 1 constructor + 1 destructor + n assignments
- Approach B: n constructors + n destructors

For classes where assignment is cheaper than constructor/destructor pair,
approach A is generally more efficient. Otherwise, approach B is probably
better. Default to approach B unless (1) assignment is cheaper than
ctor/dtor pair and (2) you're dealing with performance sensitve area of
code.

> [!abstract] Summary  
> Postpone variable definitons as long as possible. It improves clarity
> and efficiency.

### **Item 27:** Minimise casting.  
Rules of C++ are designed to guarantee that type errors are impossible.
However, casts subvert the type system. They should not be taken
lightly.

Old C style casts:
- `(T) expression`
- `T(expression)`
There is no difference in the meaning between these forms.

New C++ casts
- const_cast< T >(expression);
- dynamic_cast< T >(expression);
- reinterpret< T >(expression);
- static_cast< T >(expression);

Each serves a distinct purpose:
- `const_cast`: cast away constness. The only C++ cast that can do this.
- `dynamic_cast`: Primarily for "safe downcasting" i.e. whether an
object is of a particular type in an inheritance hierarchy. e.g. whether
`shape` can be cast to `triangle`. Cannot be performed in old style cast and
has significant runtime cost.
- `reinterpret_cast`: intended for low-level casts that yield
implementation-dependent (unportable) results. e.g. pointer to int. Should
rarely be used outside of low level code.
- `static_cast`: Used to force implicit conversions (e.g. non-const to
const. int to double etc). It can also be used to perform the reverse
of many such conversions (e.g. void pointers* to type pointers,
pointer-to-base to pointer-to-derived). Just not const to non-const.

New style casts are preferrable as they are easier to identify in code and
their narrower usage provides more helpful usage errors.

==Many programmers believe that casts do nothing but tell compilers to
treat one type as another==. This is wrong. Type conversions of any kind
can lead to code that is executed at runtime.

```cpp
int x, y;
...
double d = static_cast<double>(x) / y;
```
This code almost certainly generates code as the underlying representation
of a double differs from an int on most architectures. Not too surprising
but this may be a bit more unexpected:

```cpp
class Base {...};
class Derived : public Base {...};
Derived d;
Base *pb = &d;
```
We're creating a base class pointer to a derived object, but sometimes
the two pointer values will not be the same. When this happens, an offset
is ==*applied at runtime*== to the Derived* pointer to get the correct Base*
pointer value.

The last example demonstrates that a single object might have more than
one address (e.g. address when pointed by Base* and address when
pointed by Derived*). This does not happen in C, Java or C#, but it does
happen in C++. It happens all the time when multiple inheritance is in use.
But it can also happen in single inheritance. ==This means that you should
generally avoid making assumptions about how things are laid out in
C++ and certainly not make casts based on such assumptions.==

> [!info] Note  
> An offset is 'sometimes' required. Compiler specific implementation
> differs. Just because a cast based on assumed layout works on one
> platform does not mean it will work on another.

Casts make it easy to write code that looks right, but is in fact wrong.
For example, often virtual member function implementations are required
to call their base class counterparts first:
```cpp
class Window {
public:
    virtual void onResize() {...}
    ...
};

class SpecialWindow: public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();
        ...
    }
    ...
};
```
In this code, it may appear to be casting `this`, the current object, into its
base class and then calling `onResize()` on it. However, what it does is
creates a new, ==temporary copy of the base class== part of `this`, calls
`Window::onResize` on the copy of the base class part and continues with
the function and performs the `SpecialWindow` specific actions on the
current object.

The object might now be in an invalid state where the base class
modifications have not been made but the derived class parts have been.
To handle this properly you should do the following:
```cpp
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        Window::onResize();
        ...
    }
    ...
};
```

If you are finding yourself wanting to cast, you could be approaching
things in the wrong way.

`dynamic_cast` is an extremely costly cast. One common implementation
is partly based on string comparisons between class names. In an
inheritance hierarchy of four levels, this would cost 4 calls to `strcmp`. Be
wary of casts in general, but especially wary of dynamic_casts.

The need for dynamic_cast usually arises when you want to perform
derived class operations on what you think is a derived class, but what
you have is a pointer-to-base.

There are two possible solutions to avoid dynamic_casting for this reason:
- Use containers that store pointers to derived class objects directly.
- Use virtual functions in the base class that will allow you to do what
you need.

> [!abstract] Summary  
> - Avoid casts whenever practical, especially dynamic_casts. Explore
> cast-free alternatives if a design requires casting.
> - If casting is necessary, hide it inside a function.
> - Prefer C++ style casts to old-style casts. Easier to spot and
> more specific about what they do.


### **Item 28:** Avoid returning "handles" to object internals.  
Consider an implementation of a rectangle class that stores the upper-left
corner and lower-right corner. Tok keep the `Rectangle` object small, the
points that define it's extent aren't stored in the `Rectangle` itself, but
an auxiliary struct that the rectangle points to.
```cpp
class Point {
public:
    Point(int x, int y);
    ...
    void setX(int newVal);
    void setY(int newVal);
};

struct RectData {
    Point ulhc;
    Point lrhc;
};

class Rectangle {
    ...
private:
    std::shared_ptr<RectData> pData;
}
```

Because the clients of `Rectangle` need to be able to determine the extents
of a rectangle, there are functions `upperLeft` and `lowerRight`. Because
`Point` is a user-defined type and it is more efficient to return a reference,
these functions return references.
```cpp
class Rectangle {
public:
    ...
    Point& upperLeft() const { return pData->ulhc; }
    Point& lowerRight() const { return pData->lrhc; }
    ...
};
```
The design will compile, but it's wrong. The functions are declared const
so they are designed to offer clients a way to learn what the points of the
rectangle are, not to change them. However, since references to internal
data members are returned, they can now be changed.

```cpp
Point coord1(0, 0);
point coord2(100, 100);

const Rectangle rec(coord1, coord2); // rectangle 0,0 x 100,100

rec.upperLeft().setX(50);  // now rec 50,0 x 100,100
```

Two lessons from this example:
- ==A data member is only as encapsulated as the most accessible
function returning reference to it==
  - In this example, `ulhc` and `lrhc` are effectively public.
- ==If a const member function returns a reference to data associated
with an object that is stored outside the object itself, the called can now
change that data.==

The example involved references but the same would be true for pointers
and iterators as they all act as *handlers*.

We generally think of an object's internals as it's data members ,but 
member functions not accessible to the general public are also part of
an object's internals. This means that you should nver have a member
function return a pointer to a less accessible member function.

The rectable example can be simply solved by adding const to their return
types:
```cpp
class Rectangle {
public:
    ...
    const Point& upperLeft() const { return pData->ulhc; }
    const Point& lowerRight() const { return pData->lrhc; }
    ...
};
```
The clients can now 'read' the points but not 'write' into them.
However, `upperLeft` and `lowerRight` are still returning 'handles' to an
object's internals and this can be problematic still. In particular, it can
lead to 'dangling handles'. Consider:  
```cpp
class GUIObject{
...
};

// returns a rectangle by value
const Rectangle boundingBox(const GUIObject& obj);

GUIObject *pgo;
...
// get a ptr to upper left point of bounding box
const Point *pUpperLeft = &(boundingBox(*pgo).upperLeft());
```
Here the call to `boundingBox` returns a rectangle by value. `upperLeft` is
then called on this rectangle and the pointer to this `Point` is assigned to
`pUpperLeft`. Unfortunately, the temporary rectangle object will be
destroyed. That will then also leave `pUpperLeft` pointing to a `Point` that
no longer exists.

> [!abstract] Summary  
> - Avoid returning handles (references, pointers, or iterators) to
> object internals. Doing so maximises encapsulation, helps const
> member functions act const, and minimises the creation of dangling
> handlers.

### **Item 29:** Strive for exception-safe code.  
There are two requirements for exception safety:
- **Leak no resources**
- **Don't allow data structures to be corrupted**

Consider a class for representing GUI menus with background images. The
class is designed to be used in threaded environments so it has a mutex.
```cpp
class PrettyMenu {
public:
    ...
    void changeBackground(std::istream& imgSrc);
    ...
private:
    Mutex mutex;
    Image* bgImage;
    int imageChanges;
}
```

Now here is a possible implementation of the `changeBackground` function:
```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    lock(&mutex);                //acquire mutex
    delete bgImage;              //get rid of old bg
    ++imageChanges;              //update image change count
    bgImage = new Image(imgSrc);  //install new bg
    unlock(&mutex);              //release mutex
}
```

This function fulfils none of the requirements for exception safety. It leaks
resources as if `new Image(imgSrc)` throws, the mutex is never unlocked.
As for data structure corruption, if `new Image` throws, `bgImage` is now
pointing to a deleted object and also `imageChanges` has been incremented
when the new image has not actually been installed.

*Addressing the resource leak*. To do this we just have to use objects to
help us manage resources.
```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex);   // item 14. ensures release

    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

> [!tip] info  
> Another good thing about resource management classes is that they
> usually make functions shorter.

*Now for the data structure corruption.* We now have a choice. Exception
safe functions offer ==one of three guarantees.==
- **Basic guarantee.** If an exception is thrown, everything in the program
remains in a valid state. No objects or data structures corruped and
internal state is consistent. e.g. for `changeBackground` the original
background is kept or a default background is used.
- **Strong guarantee.** If an exception is thrown, the state of the program
is left unchanged. The functions are ==atomic==. If they succeed, they 
succeed they succeed completely, and if they fail ,the program state is
is as if it was never called at all. Functions offering the 'strong'
guarantee are easier to work with as after they are called there is only
two possible states: same as before or expected state after successful
completion. With the 'basic' guarantee the function could leave the
program in any valid state.
- **The nothrow guarantee**. Never throw exceptions as these functions
always do what they promise to. All operations on built-in types are
nothrow. Functions with an empty exception specification *do not* imply
that they are nothrow. All it says is that if the function does throw an
exception, it is a serious error.

Exception-safe code must offer one of the three guarantees from above.
It is not exception-safe if it does not. The choice is only about which
level of exception-safe code to offer as write exception-unsafe code
will cause resource leaks and corrupt data structures.

The general rule is to offer the strongest guarantee that is practical.
Although nothrow guarantee is great, it is difficult to climb out of just the
C part of C++ whilst guaranteeing it.

*Back to the changeBackground example.* We can do two very simple
things to almost provide the strong guarantee.
```cpp
class PrettyMenu {
    ...
    std::shared_ptr<Image> bgImage;
    ...
};
void PrettyMenu::changeBackgroun(std::istream& imgSrc)
{
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
```
The two improvements are:
- Using a shared_ptr to avoid resource leaks.
- Reorder the statements so that `imageChanges` is only incremented until
the image has actually been changed.

The improvements make it 'almost' strong as it is possible that the
read marker for input stream 'imgSrc' has been moved.

There is a general design strategy that typically leads to the strong
guarantee. ==Copy and swap==. Make a copy of the object you want to
modify, make the necessary modifications to the copy. If an exception
is thrown, the original is left unchanged. After all changes have been
made, swap the modified object with the original in a non-throwing
operation.

This is usually implemented by putting all the per-object data into a
separate implementation object, then giving the real object a pointer
to its implementation object. This is called the "pimpl idiom".

The ==copy-and-swap== strategy is an excellent way to make all-or-nothing
changes to an object's state, but still does not guarantee that the overall
function is strongly exception safe.

```cpp
void someFunc()
{
    ...    //make copy of local state
    f1();
    f2();
    ...    //swap modified state into place
}
```

If `f1` or `f2` are less than strongly safe, `someFunc` is almost certainly not
strongly safe. But even if they are, `f1` could succeed and `f2` could fail.
This means that the state is not the same as if `f1` and `f2` both ran all the
way successfully. Such issues can prevent you from offering the strong
guarantee.

Another issue with 'copy and swap' is efficiency. The strategy requires
creating a copy of each object to be modified which takes extra time and
space.

If you are unable to provide the strong guarantee because of limitations
such as these, it is perfectly ok to only provide the weak guarantee. Just
make an effort to provide the strong guarantee.

> [!abstract] Summary  
> - Exception-safe functions ==leak no resource== and ==do not allow
> data structure corruption==. Even if exceptions are thrown; basic,
> strong, or nothrow guarantees are provided.
> - Strong guarantee can be provided via copy-and-swap.
> - A function can usually offer a guarantee no stronger than the
> function it itself calls.

### **Item 30:** Understand the ins and outs of linlining.  
Inline functions are a great idea: look like functions, act like functions,
but without the overhead of a real function call and without using macros.
==What more could you ask for?==  
They provide even more benefits! Compiler optimisations also work better
as they are typically designed for long stretches of code that lack function
calls.

However, nothing in life is free. They increase the size of your object
code. This could be a problem on machines with limited memory. Even
with virtual memory, code bloat can lead to additional paging, a reduced
instruction cache hit rate and the performance penalties these incur.

On the other hand, if an inline function body is very short, the code
generated for the function body may be smaller than the code for a
function call. This can lead to *smaller* object code and a higher instruction
cache hit rate.

Bear in mind that `inline` is a 'request' not a command. The request can
be explicit or implicit. The implicit method is to define the function in the
class definition.
```cpp
class Person {
public:
    ...
    int age() const { return theAge; }  //implicit inline req
    ...
private:
    int theAge;
};
```

The explicit way is to precede its definition with the `inline` keyword.
```cpp
inline const T& std::max(const T& a, const T& b)
{ return a < b ? b : a; }
```

Just because inline functions and function templates are usually defined
in header files does **not** mean that function templates must be inline. The
reason that this happens is because inlining and function template
instantiation both happens at compile time. If you inline all function
templates mindlessly, you will bloat the code unnecessarily and incur all of
the associated costs.

*Inline is a request.* Most compilers refuse to inline functions that are too
complicated (contain loops or are recursive) and all but the most trivial
calls to virtual functions. After all virtual means 'figure out which
function to call at runtime'. How you call the function also affects
whether the function is inlined. Calling a function through a pointer will
most likely not result in it being inlined.

It is also important to carefully consider inlining as a library designer as it
is impossible to provide binary upgrades to the client-visible inline
functions in a library. This is because inline functions are compiled into
the clients application. To change this the clients would have to recompile
with the new provided inline function code. This is more onerous than
dynamically linking a non inline function.

This all culminates in the logical strategy that functions should generally
not be declared inline. The 80-20 rule should be recalled and you should
identify the 20% of your code that increases your code's performance
and selectively apply inline.

> [!abstract] Summary  
> - Limit most inlining to small, frequently called functions. This
> facilitate debugging, binary upgradability, minimises code bloat, and
> maxmises chances of greater performance.
> - Don't declare function templates inline just because they appear
> in header files.

### **Item 31:** Minimise compilation dependencies between files.  
C++ does not do a very good job of separating interfaces from
implementations. You might make a small change to an implementation,
just the private stuff and find your whole project recompiling.

```cpp
class Person {
public:
    ...
    Person(const std::string& name, const Date& birthday,
           const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::string theName;  //implementation detail
    Date theBirthDate;    //implementation detail
    Address theAddress;   //implementation detail
};
```

In this example, the `Person` class can't be compiled without access to
definitions that the `Person` ==implementation== uses (string, Date, Address).
Such definitions are typically provided using the `#include` directive.
```cpp
#include <string>
#include "date.h"
#include "address.h"
```

Unfortunately, this leads to compilation dependencies as changes to any
of these files will require Person to also be recompiled. C++ requires the
implementation details of a class in the class definition as compilers need
to know how much memory to allocate for objects. This info cannot be
found in forward declarations. ==This is not a problem in languages like
Java== as in those languages compilers allocate only enough space for a
pointer to an object.

In C++ there are two ways to solve this problem:
- Like Java, play the game of "hide the object implementation behind a
pointer game" using the pimpl idiom.
- Use ==interface classes==.

*pimpl idion method*
```cpp
#include <string>
#include <memory>

class PersonImpl;
class Date;
class Address;

class Person {
    Person(const std::string& name, const Date& birthday,
           const Address addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::shared_ptr<PersonImpl> pImpl;//ptr to implementation
};
```

With this design, clients of Person are divorced from the details of dates,
addresses and persons. The implementations can be changed at will, but
`Person` clients do not need to recompile. ==Furthermore, since the clients
are unable to view the implementation, the are unlikely to write code that
depends on those details==. Hence:
- *Avoid using objects when object references will do*. You can define
pointers and references to a type with only a declaration for a type.
Defining objects of a type is what necessitates the definition of the class.
- *Depend on class declarations instead of class definitions whenever
you can*. You never need a class definition to declare a function using that
class, not even if the function passes or returns the class type by value.
```cpp
class Date;   //class declaration

Date today(); // fine even though it returns Date by value
void clearAppointments(Date d); // also fine
```
Why declare functions that no one calls? It allows you to have a header
file of function declarations. However, clients of your library will most
likely not use every function. Now the burden of `#include`'ing the types
the clients need is on the client. If you put all of the includes in the
function declaration file, the clients would have artificial dependencies on
files that they do not need as they would be including things that they
don't need from the functions they don't call.
- *Provide separate header files for declarations and definitions.* To
facilitate adherance to these guidelines, header files should come in pairs:
one for declarations, the other for definitions. These files need to be kept
consistent. This means that library clients should `#include` a declaration
file instead of forward declaring themselves. This means that library
authors should provide both of these header files. The above example
would look like so:
```cpp
#include "datefwd.h"

Date today();
void clearAppointments(Date d);
```

Classes like `Person` that use the 'pimpl idiom' are often called handle
classes. Here's how `Person` might be implemented(aka the .cpp file):
```cpp
#include "Person.h"
#include "PersonImpl.h"

Person::Person(const std::string& name, const Date& birthday,
               const Address& addr)
:pImpl(new PersonImpl(name, birthday, addr))
{}

std::string Person::name() const
{
    return pImpl->name();
}
```

Now lets consider the alternative. Interface classes. An interface class
for `Person` might look like this:
```cpp
class Person {
public:
    virtual ~Person();

    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;
}
```

==Clients of this class must program in terms of Person pointers and
references as it is impossible to instantiate classes containing pure virtual
functions.== It is however, possible to instantiate classes *derived* from
`Person`. Like handle classes, clients of the interface class do not have to
recompile unless the interface is modified.

Clients of interface classes need a way to create new objects. This is
typically done using 'factory functions' that play the role of the
constructor. They return pointers (preferably smart) to dynamically
allocated objects that support the Interface classes interface. Such
functions are often declared static inside the interface class:
```cpp
class Person {
public:
    ...
    static std::shared_ptr<Person>
        create(const std::string& name,
               const Date& birthday,
               const Address& addr);
    ...
};

// to use the factory function
std::string name;
Date dob;
Address addr;

std::shared_ptr<Person> pp(Person::create(name, dob, addr));
```

At some point the Person interface class will need to be 'implemented'.
That might look like this:
```cpp
class RealPerson: public Person {
public:
    RealPerson(const std::string& name, const Date& dob,
               const Address& addr)
    : theName(name), theBirthdate(dob), theAddress(addr)
    {}
    virtual ~RealPerson() {}
    virtual std::string name() const;
    virtual std::string birthDate() const;
    virtual std::string address() const;
private:
    std::string theName;
    Date theBirthDate;
    Address theAddress;
};

...
shared_ptr<Person> Person::create(const std::string& name,
                                  const Date& birthday,
                                  cosnt Address& addr)
{
    return std::shared_ptr<Person>(new RealPerson(name,dob
                                                  addr));
}
```

This demonstrates one of two most common mechanisms for implementing
an Interface class: inherit its interface specification, then implement the
function from the interface. A second way is using multiple inheritance
(item 40).

==So what are the costs for all this magic?== Some speed at runtime, plus
some additional memory per object. For handle classes, you have to go
through one layer of indirection to get to the object's data per access.
The space required for this extra implementation pointer increases the
memory required as well. Finally, since the object is dynamically
allocated, there are associated costs with that.

For interface classes, every function call is virtual, so you pay the cost
of an indirect jump each time you make a function call. All objects derived
from an interface class also requires a virtual table pointer.

Finally, for both, they don't really benefit from `inline` functions.

Do not dismiss ==handle classes== and ==interface classes== due to these costs
though. Use them evolutionarily. Use them during development and then
replace with concrete classes for production use if you can prove the
speed/size benefit outweighs the coupling.

> [!abstract] Summary  
> - Main idea to reduce compilation dependencies is to rely on
> declarations as much as possible. The two approaches are =='handle
> classes'== and =='interface classes'==.
> - Library header files should exist in full and declaration-only forms
> even if templates are involved.

## Ch6: Inheritance and Object-Oriented Design  

### **Item 32:** Make sure public inheritance models "is-a."  
The most important rule with OOP in C++ is that *public inheritance means
"is-a"*. If you write that class D(derived) publicly inherits from class
B(base), you are telling C++ compilers that every object of type D is also
an object of type B. Anywhere an object of type B can be used, you can
also use type D, but not vice versa.
```cpp
class Person {...};
class Student: public Person {...};

void eat(const Person& p);
void study(const Student& s);

Person p;
Student s;

eat(p);  //fine - p is a person
eat(s);  //fine - s is a student. student is-a person

study(s);//fine
study(p);//error - p is not a student. 
```

This is true only for *public* inheritance. Private inheritance means
something different. Who knows what protected inheritance means.

However, human intuition can be misleading. If we say penguins are birds
and birds can fly, we could represent it like this:
```cpp
class Bird {
public:
    virtual void fly();      //birds can fly
    ...
};
class Penguin: public Bird { // Penguins are birds
    ...
};
```

Now, we are in trouble because this hierarchy is claiming that penguins
can fly. We are victims of an imprecise language called english. When we
say birds can fly, we don't mean *all* birds can fly. There can be non flying
birds. Now that we are clear about this, we could represent it like so:
```cpp
class Bird {
public:
    ...                      // no fly function
};
class FlyingBird {
public:
    virtual void fly();
    ...
};
class Penguin: public Bird { // Penguins are birds
    ...
};
```
This is now more faithful to reality. However, if your application does not
deal with flying, the original two class hierarchy may be sufficient. It
depends on what the system is expected to do.

There is another school of thought on how to handle the "penguins are
birds but they can't fly" issue. The approach is to redefine the fly
function for penguins so that it generates a *runtime* error.  
```cpp
void error(const std::string& msg); //defined elsewhere

class Penguin: public Bird {
public:
    virtual void fly() { error("Penguins don't fly!");}
    ...
};
```
It is important to recognise that this does *NOT* say "penguins can't fly". It
actually says "Penguins can fly, but it's an error when they actually try
to do so". The main way to detect this is that the error is actually only
detected at *runtime*. The way to correctly, definitively say that penguins
can't fly is to not implement the fly function for penguins so that now the
compiler will detect the error at *compile* time.

Lets consider another example. Should square inherit from rectangle? You
might be tempted to think so. But you might run into this issue:
```cpp
class Rectangle {
public:
    virtual void setHeight(int newHeight);
    virtual void setWidth(int newWidth);
    virtual int height() const;
    virtual int width() const;
    ...
};
void makeBigger(Rectangle& r)
{
    int oldHeight = r.height();
    r.setWidth(r.width() + 10);
    assert(r.height() == oldHeight); //ensure height unchanged
}

class Square: public Rectangle {...};
Square s;
...
assert(s.width() == s.height());  //true for squares. surely
makeBigger(s); // is a rectangle by inheritance so can call it
assert(s.width() == s.height());  //true for squares. surely
```
It is clear that the second assertion on the square should remain true as
a square by definition has equal height and width. However, the call to
`makeBigger()` has changed the width but not height. The fundamental
issue here is that something applicable for rectangles is not applicable for
squares. This property is: ==height can be changed independently of width==.
But public inheritance asserts that everything that applies to the base
applies to the derived class too. 

> [!abstract] Summary  
> Public inheritance means "is-a". Everything that applies to a base
> must also apply to a derived class because a derived class is-a base
> class.

### **Item 33:** Avoid hiding inherited names.  
```cpp
int x;
void someFunc()
{
    double x;
    std::cin >> x;
}
```
From the above code it is easy to recognise that global variable x is hidden
by the `x` in the inner scope. Inheritance behaves similarly as if the scope
of the derived class is inside a base class.

```cpp
class Base {
private:
    int x;
public:
    virtual void mf1() = 0;
    virtual void mf1(int);

    virtual void mf2();

    mf3();
    void mf3(double);
    ...
};

class Derived: public Base {
public:
    virtual void mf1();
    void mf3();
    void mf4();
};
```
In this example, `mf1` and `mf3` hide the base class version. From the
perspective of *name lookup*, Base::mf1 and Base::mf2 are no longer
inherited at all.

```cpp
Derived d;
int x;
...
d.mf1(); //fine. calls Derived::mf1
d.mf1(x) //error. Derived::mf1 hides Base::mf1
d.mf2(); //fine. calls Derived::mf2
d.mf3(); //fine. calls Derived::mf3
d.mf3(x);//error. Derived::mf3 hides Base::mf3
```

The surprising behaviour is that the hiding also affects functions in the
base class that take different parameters. The rationale is that it prevents
accidentally inheriting overlods from distant base classes when you create
a new derived class in a library or application framework.

Unfortunately, you do want to inherit the overloads when using public
inheritance. You don't wan to violate the *"is-a"* relationship. You can
override this behaviour with `using declarations`.

```cpp
class Derived: public Base {
public:
    using Base::mf1; //make things in Base called mf1 and
    using Base::mf1; //mf1 visible in Derived's scope

    virtual void mf1();
    void mf3();
    void mf4();
};
Derived d;
int x;
...
d.mf1(); //fine. calls Derived::mf1
d.mf1(x) //now ok. Calls Base::mf1
d.mf2(); //fine. calls Derived::mf2
d.mf3(); //fine. calls Derived::mf3
d.mf3(x);//now ok. Calls Base::mf3
```

In public inheritance you always want to inherit all the functions from the
Base class. However, with private inheritance, you may not want to
inherit all the functions. Lets say Derived privately inherits from Base, and
it only wants to inherit the `mf1` that takes on parameters. You can't use
`using` to do this. For this you have to use a ==forwarding function==.

```cpp
class Base {
public:
    virtual void mf1() = 0;
    virtual void mf1(int);
    ...
};
class Derived: private Base {
public:
    virtual void mf1() { Base::mf1(); }
    ...
};
...
Derived d;
int x;
d.mf1();  //works as expected
d.mf1(x); //hidden as expected
```

> [!abstract] Summary  
> - Names in derived classes hide names in base classes. Under pubilc
> inheritance, this is never desirable.
> - To make hidden names visible again, employ using declarations or
> forwarding functions.


### **Item 34:** Differentiate between inheritance of interface and inheritance of implementation.  
Public inheritance is composed of two separable parts: inheritance of
function interfaces and inheritance of function implementations. This
corresponds exactly to the difference between function declarations
and function definitions.

As a class designer you may want to (1) allow a derived class to only
inherit the interface of a member function, (2) allow a derived class to
inherit a function's interface and implementation, but with the option to
override the inherited implementation or (3) allow a derived class to
inherit interface and implementation without allowing it to be overriden.

#### **1)** Interface only - AKA pure virtual functions
```cpp
class Shape {
public:
    virtual void draw() const = 0;
    ...
};
```
Two most salient features of pure virtual functions:  
- Must be redeclared by conrete class that inherits them.
- Typically have no definition in abstract classes.
These two imply that the whole point of pure virtual functions is to have a
derived class inherit the *interface only*.

In the shape example, we have no idea how a shape will be drawn. Just
that the designer of a concrete shape derived class has to make provide
an implementation.

#### **2)** Interface + optional default implementation - AKA virtual functions
The big risk with virtual functions is that a new derived class is added into
the inheritance hierarchy, *BUT* this new derived class differs from its
siblings and does *NOT* want the default implementation of the virtual
function. If the programmers forget to implement the custom overriden
version for this new derived class, then we have issues.

What we would like to do is offer a default implementation but require it to
be explicitly asked for instead of being given it by default.  
There are two ways to solve this. The first:
```cpp
class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    ...
protected:
    void defaultFly(const Airport& destination);
};

void Airplane::defaultFly(const Airport& destination)
{
    // default code to fly to desination airport
    ...
}
```
The key here is to disconnect the interface from the default
implementation. The fly virtual function has been changed into a pure
virtual function which has to be implemented but the implementation can
now just use the `defaultFly` function provided.

Now any new derived classes will have to implement `fly` and will not use
the `defaultFly` function accidentally.

An objection to this method of solving the problem is the pollution of the
namespace. You can solve this with something very similar:
```cpp
class Airplane {
public:
    virtual void fly(const Airport& destination) = 0;
    ...
};

void Airplane::fly(const Airport& destination)
{
    // default code to fly to destination
    ...
}

class ModelA: public Airplane { // uses default  fly
public:
    virutal void fly(const Airport& destination)
    {Airplane::fly(destination);}
    ...
};

class ModelB: public Airplane {
public:
    virutal void fly(const Airport& destination);
    ...
};

void ModelB::fly(const Airport& destination)
{
    // code for ModelB plane to fly to destination
}
```
The main thing you lose with this method is to have different protection
levels. The code that used to be in `defaultFly` is now public.

#### **3)** Interface + mandatory implementation
```cpp
class Shape {
public:
    int objectID() const;
    ...
}
```
Nothing fancy in terms of implementation. But you should consider the
implication of a mandatory implementation. It's basically saying "Every
shape object has a funtion that returns it's ID, and no derived class can
change the implementation". It is an ==invariant over specialisation==.

**Common mistakes**
- Making everything non-virtual
    - Might happen due to concerns over performance. Just remember the 80-20 rule.
- Declaring all member functions virtual
    - Can be sign of a class designer to make a firm stance on what functions really are invariants.

> [!abstract] Summary  
> - Inheriting interfaces is different from inheriting implementations.
> Under public inheritance, derived classes always inherit base class
> interfaces.
> - Pure virtual functions specify inheritance of interface only.
> - Simple (impure) virtual functions specify inheritance of interface
> and inheritance of a default implementation.
> - Non-virtual functions specify inheritanc of interface and a
> mandatory implementation.


### **Item 35:** Consider alternatives to virtual functions.  
Consider a video game and a hierarchy for characters in the game. You
offer a member function `healthValue`, that returns an integer indicating
how healthy the character is. However, different characters may calculate
their health differently. Declaring `healthValue` virtual seems obvious:
```cpp
class GameCharacter {
public:
    virtual int healthValue() const;
    ...
};
```
The problem is in fact that this design does seem so obvious. However,
there are alternative ways to approach this problem and we should bear
this in mind and consider them.

#### The Template Method Pattern via the `Non-Virtual Interface Idiom`  
One school of thought is that virtual functions should always be private.
To adhere to this rule and solve the healthvalue problem you could retain
`healthValue` but change it to public. Then a different private virtual
function does the actual calculation and is called by the public function:
```cpp
class GameCharacter {
public:
    int healValue() const
    {
        ...  //do "before" stuff. aka setup
        int retVal = doHealthValue(); //do real work
        ...  //do "after" stuff. aka cleanup(?)
        return retVal;
    }
    ...
private:
    virtual int doHealthValue() const
    {
        ... //default algorithm for health calc
    }
};
```
This design is calle dthe non-virtual interface (NVI) idiom. It is a
minifestation of a more general design pattern called Template Method
(unrelated to C++ templates).

in the NVI idiom, the non-virtual method becomes the 'wrapper' used to
call the virtual method that does the work. This means that the wrapper
function can ensure that the context is setup before the work is done and
then cleaned up after (e.g. Locking and unlocking a mutex).

#### The Strategy pattern via Function Pointers
The NVI idiom is an interesting alternative to public virtual functions, but
ultimately it's little more than window dressing. After all, the real work
is still being done with virtual functions. A more dramatic design would be
saying that calculating a character's health is independent of the
character's type. For example we could require that each character's
constructor be passed a pointer to a health calculation function.
```cpp
class GameCharacter;  // forward declaration

//function for default health calculation algo
int defaultHealthCalc(const GameCharacter& gc);

class GameCharacter {
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
    : healthFunc(hcf)
    {}
    int healthValue() const
    { return healthFunct(*this); }
    ...
private:
    HealthCalcFunc healthFunc;
};
```

This approach is a simple application of another common design pattern -
==strategy==. Compared to approaches based on virutal functions, it offers
some interesting flexibility:
- Different instances of the same character type can have different health
calculation functions.
- Health calculation functions can be changed at runtime.

However, on the other hand, it means that the health calculation function
no longer has special access to the internal parts of `Character`. This
is something that you have to bear in mind for all functions outside of the
class. The only way to resolve this is to weaken the encapsulation of the
class i.e. declare non-member function a friend, offer public accessor
functions that would otherwise not be necessary.

#### The Strategy Pattern via std::function
Once you get accustomed to templates though, the function pointer
approach can seem limiting and rigid. Why can't we use something that
*acts* like a function such as a function object? Why can't it be a member
function? Why must it return an int instead of a type convertible to an int.

These constraints vanish when we use std::function instead of a function
pointer. Here is the same design but with std::function.
```cpp
class GameCharacter;
int defaultHealthCalc(const GameCharacter& gc);

class GameCharacter {
public:
    typedef std::function<int (const GameCharacter&)> HealthCalcFunc;

    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc)
    :healthFunc(hcf)
    {}
    int healthValue() const { return healthFunc(*this);}
    ...
private:
    HealthCalcFunc healthFunc;
};
```

Lets take a closer look at what the typedef was for:  
`std::function<int (const GameCharacter&)>`  
The target signature is "function taking a const GameCharacter& and
returning an int". An object of this type can hold any callable entity
compatitble with that signature. Compatible means that
`const GameCharacter&` can be converted to the type of the entity's
parameter and the entity's return type can be converted to int.

This is similar to using function pointers but with a staggering amount
more flexibility.

#### The "Classic" Stretegy Pattern
For a more design pattern friendly approach instead of a 'C++ coolness'
approach you can have `GameCharacter` be the base class for `EvilBadGuy`
and `EyeCandyCharacter`. `HealthCalcFunc` would then be the root of
another hierarchy with derived classes `SlowHealthLoser` and
`FastHealthLoser`. Each `GameCharacter` would contain a pointer to an
object from the `HealthCalcFunc` hierarachy.
```cpp
class GameCharacter;

class HealthCalcFunc {
public:
    ...
    virtual int calc(const GameCharacter& gc) const
    {...}
    ...
};

HealthCalcFunc defaultHealthCalc;

class GameCharacter {
public:
    explicit GameCharacter(HealthCalcFunc* phcf = &defaulthealthCalc)
    : pHealthCalc(phcf)
    {}
    int healthValue() const
    { return pHealthCalc->calc(*this);}
    ...
private:
    HealthCalcFunc *pHealthCalc;
};

```

This approach has the appeal of being quickly recognisable to people
familiar with the 'standard' strategy pattern implementation. It also has
the benefit of allowing the existing health calculation to be tweaked with
the use of derived classes.

> [!abstract] Summary  
> Consider alternatives to virtual functions when searching for a design
> to solve your problem. Alternatives include:
> - **Non-virutal Interface Idiom**. A form of the Template Method design
> - Replace virtual functions with **function pointer data members**
> - Replace virtual functions with **std::function data members**
> - Replace virtual functions in one hierarchy with **virtual functions from a different hierarchy**.
> This is the conventional approach to the strategy pattern.

### **Item 36:** Never redefine an inherited non-virtual function.  
Lets say you have a base class B and a derived class D that inherits from
B.
```cpp
class B {
public:
    void mf();
    ...
};
class D: public B{...}
```
Now if you were to call mf though a pointer to object D...
```cpp
D x;
B* pB = &x;
D* pD = &x;

pB->mf();  // surely these function calls
pD->mf():  // do the same thing
```
You would expect the same result. However, this may not be the case if
class D has redefined `mf()`. This is because ==non-virtual functions are statically bound==.
Virtual functions. on the other hand are dynamically bound, so you would
get the expected result. That's the pragmatic argument.

Now for a more theoretical argument. If public inheritance means an "is-a"
relationship and declaring a non virtual function establishes an invariant
over specialisation for that class, then:
- Everything that applies to B objects also apply to D objects
- Classes derived from B must inherit both the interface and
implementation of `mf()` because mf is non-virtual

Then what is D do doing by redefining `mf()`? It makes no sense. It means
that every D is a B is no longer true. It would mean that D should not
inherit from B. If D does really have to inherit from B, then that also
means that the invariant over specialisation is not true. This means that
`mf()` should be virtual.

Basically, just don't do it. It's pragmatically bad and theoretically
nonsensical.

> [!abstract] Summary  
> Never redefine an inherited non-virtual function


### **Item 37:** Never redifine a function's inherited default parameter value.  
The previous item rules out redefining non virtual functions so this item
address redefining the *default parameter value* of a virtual function.

==Virtual functions are dynamically bound, but default parameter values are statically bound.==

Essentially this means that if you redefine a default parameter value in a
derived function, it will still use the default parameter value from the base
class. So don't do it.

If you want to supply a default parameter value for both the base and
derived class though, without redefining the same colour in both, resulting
in code duplication and dependencies, you can use NVI.

> [!abstract] Summary  
> Never redefine an inherited default parameter value because default
> parameter values are statically bound and virtual functions are
> dynamically bound.

### **Item 38:** Model "has-a" or "is-implemented-in-terms-of" throw composition.  
*Composition* is the relationship that arises when objects of one type
contain objects of another type. For example:
```cpp
class Address {...};
class PhoneNumber {...};
class Person {
public:
    ...
private:
    std::string name;         //composed object
    Address;                  // ditto
    PhoneNumber voiceNumber;  // ditto
    PhoneNumber faxNumber;    // ditto
};
```

Item 32 explains that public inheritance means "is-a". Composition also
has a meaning. Composition means "has-a" or
"is-implemented-in-terms-of". That's because some objects in your code
correspond to things in the world you are modeling e.g. people, vehicles,
video frames etc. Such objects are part of the *application domain*. Other
objects are purely implementation artefacts e.g. buffers, mutexes, search
trees etc. Application domain -> "has-a". Implementation domain ->
"is-implemented-in-terms-of".

The "has-a" relationship is relatively simple and people do not struggle
with it. More problematic is the "is-implemented-in-terms-of"
relationship. Lets say you need a hash set. The STL has such a data
structure. However, the tradeoff made in the STL implementation is
to sacrifice space for speed and requires three pointers per element. So
you decided to implement as set yourself. You decide to implement it as
a linked list and not a search tree. To do this you decide to use the STL
implementation of linked list. You may choose to do the following:
```cpp
template<typename T>
class Set: public std::list<T> { ... };
```

This would be wrong. This implies that what's true for list is also true
for your set. It models an "is-a" relationship. what you should do is:
```cpp
template<typename T>
class Set {
public:
    bool member(const T& item) const;
    void insert(const T& item);
    void remove(const T& item);
    std::size_t size() const;
private:
    std::list<T> rep;     // representation of set data
};
```

Finally the relationship has been established correctly. Set is "implemented
in terms of" a linked list.

> [!abstract] Summary  
> - Composition has meanings completely different from that of public
> inheritance.
> - In the application domain, composition means "has-a"
> - In the implementation domain, composition means "is-implemented-in-terms-of"

### **Item 39:** Use private inheritance judiciously.  
Lets repeast the 'A Person can eat and a Student can study' example from
the public inheritance section, but with private inheritance.
```cpp
class Person {...};
class Student: private Person {...};  //private inheritance
void eat(const Person& p);            // any person can eat
void study(const Student& s);         // only students study
Person p;
Student s;
eat(p);   //fine, p is a person
eat(s);   //error! a Student isn't a person
```

As you can see compilers will generally *NOT* convert a derived class into
the base class with private inheritance. This seems to indicate that it is
*NOT* an "is-a" relationship. The other thing that happens is that members
inherited from a private base class will all be converted to private
members of the derived base class even if they are public or protected.

So what is the meaning? Private inheritance means
"is-implemented-in-terms-of". If you make class D privately inherit from
class B, you are doing so to take advantage of some features in B, not
because there's any conceptual link between them. If we think of it in
a way similar to item 34, private inheritance means to *only* inherit the
implementation and not the interface.

The catch is that this 'meaning' is the exact same as item 38 -
composition. So when should you use one or the other?  
==Use composition whenever you can, and use private inheritance when you must.==
And when must you? Primarily when protected members and/or virtual
functions enter the picutre.

Lets consider a widget class that runs overtime. We want to track how
often member functions are being called in regular time intervals. We
might reuse a Timer class that looks like this:
```cpp
class Timer {
public:
    explicit Timer(int tickFrequency);
    virtual void onTick() const;  //auto called for each tick
    ...
};
```

This class is perfect as we can redefine the virtual function so that it
examines the state of the `Widget`. In order to redefine a virtual function
though we need to inherit from Timer. However, public inheritance is
not suitable as clients would then be able to call onTick which we do not
want. Therefore we inherit privately:
```cpp
class Widget: private Timer {
private:
    class WidgetTimer: public Timer {
    public:
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
    ...
};
```

This is a nice design but we need to remember that this can still be done
with composition. It would be a bit more complicated though. We would
have to declare a private nested class inside Widget that would publicly
inherit from Timer, redefine `onTick()` in there and put an object of that
type into Widget.
```cpp
class Widget {
private:
    class WidgetTimer: public Timer {
    public:
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
    ...
};
```

Showing this design is primarily a reminder that there are many ways to do
the same thing and they should be considered. However, there are two
rearsons why public inheritance + composition might be preferrable over
private inheritance.
- You might want to design Widget to allow for derived classes but
prevent derived classes from redefining `onTick`. Private inheritance would
still allow it as you can still redefine functions that you can't call. If you
want to replicate Java/C# ability to prevent derived classes from
redefining virtual functions this is a way to approximate that behaviour.
- You might want to minimise Widget's compilation dependencies. If
Widget inherits from Timer, Timer's definition must be available when
Widget is compiled so the file defining Widget probably has to
`#include "Timer.h"`. On the other hand, if WidgetTimer is moved out of
Widget and Widget contains only a pointer to a WidgetTimer, Widget can
get by with a simple declaration for `WidgetTimer`.

There is an edge case that private inheritance might be useful for involving
space optimisation. It applies when dealing with classes that have no data
in it: no non-static data members; no virtual functions; and no virtual
base classes. In theory these sort of classes should use no space.
However, they do.
```cpp
class Empty {}; //has no data so objects should use no memory

class HoldsAnInt {  // should only use space for 1 int
    int x;
    Empty e;    //should require no memory
};
```
You'll find that `sizeof(HoldsAnint) > sizeof(int);`. This is because
with most compilers, sizeof(Empty) is 1. What makes this even worse is
that alignment requirements can make HoldsAndInt take even more space.
It would take the space of 2 ints, instead of 1 int + 1 char.

This is because C++ has an edict against zero-size free standing objects.
However, this does mean that if you inherit from Empty instead, the size
will be as expected:
```cpp
class HoldsAnInt: private Empty {
private:
    int x;
};
```
Now `sizeof(HoldsAnInt) == sizeof(int)`. This is known as the 'empty
base optimisation' (EBO). In practice, most empty classes aren't emtpy,
they contain typedefs, enums, static data members, or non-virtual
members. The STL has many 'technically' empty classes. Thanks to EBO,
these classes rarely increase the size of the inheriting class.

However, this use case is indeed niche. It's good to be aware of but most
likely rarely used.

> [!abstract] Summary  
> - Private inheritance means "is-implemented-in-terms-of". It is
> usually inferior to composition, but makes sense if a derived class
> needs access to protected base class members or to redefine virtual
> functions.
> - Unlike composition, private inheritance can enable the *empty base otimisation*.
> This can be important for library developers who strive to minimise
> object sizes.

### **Item 40:** Use multiple inheritance judiciously.  
C++ community seems to break up into two camps regarding multiple
inheritance:
- if single inheritance(SI) is good, multiple inheritance(MI) must be better
- single inheritance is good, but multiple inheritance isn't worth the
trouble

First thing to recognise with multiple inheritance is that it makes it
possible to inherit the same name leading to ambiguity. For example you
might inherit from two classes that both have the `checkout()` function.
To fix this you would then have to specify which one you are calling:  
`BorrowableItem::checkout();`  
`ElectronicGadget::checkout();`  

MI can also lead to the 'deadly MI diamond'.
```cpp
class File {...};

class InputFile: public File {...};
class OutputFile: public File {...};

class IOFile: public InputFile, public OutputFile
{...};
```
Here you have to confront the question whether you want to replicate the
data members of the base class for each 'path':
- IOFile -> InputFile -> File
- IOFile -> OutputFile -> File
C++ takes no position and happily supports both option. The default is to
perform the replication. If you don't want that, you have to make the
class with the data a virtual base class:
```cpp
class File {...};

class InputFile: virtual public File {...};
class OutputFile: virtual public File {...};

class IOFile: public InputFile, public OutputFile
{...};
```

From the viewpoint of correct behaviour, public inheritance should always
be virtual. However, correctness is not the only perspective. Virtual
inheritance has space and time costs. It also has complexity costs.
==The responsibility of initialising a virtual base is borne by the most derived class in the hierarchy==.
This means that (1) classes derived from virtual base classes must be aware of their virtual bases no matter how distant and (2) when a new
derived class is added to the hierarchy, it must assume direct initialisation
responsibilities for its virtual bases.

Therefore the advice on virtual base classes is simple. Don't use virtual
bases unless you have to. If you must, then avoid putting data in them so
you don't have to worry about initialisation oddities. Interfaces in Java
are in many ways comparable to virtual base classes in C++ and are also
not allowed to contain any data.

One legitimate use is to combine public inheritance of an interface with
the private inheritance of an implementation.
```cpp
class IPerson {
public:
    virtual ~IPerson();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
};
// factory function to create Person object
std::shared_ptr<IPerson> makePerson(DatabaseID personIdentified);
```
Here we have an interface IPerson and a factory function declaration used
to create Person objects. But there has to be a concrete class for
makePerson to instantiate and return a pointer for.

Now lets assume we  already have a class called PersonInfo that we want
to reuse to fit the IPerson interface.
```cpp
class PersonInfo {
public:
    explicit PersonInfo(DatabaseID pid);
    virtual ~PersonInfo();
    virtual const char* theName() const;
    virtual const char* theBirthDate() const;
    ...
private:
    virtual const char* valueDelimOpen() const;
    virtual const char* valueDelimClose() const;
    ...
};
```

If we want to create a new class `CPerson` using the existing `PersonInfo`
class, the relationship is clearly "CPerson is implemented in terms of
PersonInfo". So we could combine private and public inheritance like so:
```cpp
class CPerson: public IPerson, private PersonInfo {
public:
    explicit CPerson(DatabaseID pid): PersonInfo(pid) {}

    virtual std::string name() const
    { return PersonInfo::theName();}
    virtual std::string birthDate() const
    { return PersonInfo::theBirthDate(); }
private:
    const char* valueDelimOpen() const { return "";}
    const char* valueDelimClose() const { return "";}
};
```

The example should demonstrate that multiple inheritance can be both
useful and sensible. However, single inheritance is typically better. Just
be judicious with the use of MI.

> [!abstract] Summary  
> - Multiple inheritance is more complex than single inheritance. It can
> lead to ambiguity issues.
> - Virtual inheritance imposes costs in size, speed, and complexity of
> initialisation and assignment. It's most practical when virtual base
> classes have no data.
> - Multiple inheritance does have legitimate uses. One scenario involves
> combining public inheritance from an iterface class with private
> inheritance from a class that helps with implementation

## Ch7: Templates and generic programming  

### **Item 41:** Understand implicit interfaces and compile time polymorphism
The world of OOP revolved around *explicit* interfaces and *runtime*
polymorphism. For example:
```cpp
class Widget {
public:
    Widget();
    virtual ~Widget();
    virtual std::size_t size() const;
    virtual void normalize();
    void swap(Widget& other);
    ...
};
// and also this function:
void doProcessing(Widget& w)
{
    if (w.size() > 10 && w != someNastyWidget) {
        Widget temp(w);
        temp.normalize();
        temp.swap(w);
    }
}
```

We can say this about `w` in `doProcessing`:
- Because `w` is declared to be of type Widget, `w` must support the Widget
interface in the source code to see exactly what it looks like. This is an
*explicit interface* - one explicitly visible in the source code.
- Because `w` is a reference and some of Widget's member functions are
virtual, `w`'s calls to those functions will exhibit *runtime polymorphism*.

Templates and generic programming bring *compile-time polymorphism* to
the fore. Look what happens when we change `doProcessing` into a
function template:
```cpp
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w != someNastWidget) {
        t temp(w);
        temp.normalise();
        temp.swap(w);
    }
}
```
Now what can we say about `w` in doProcessing?
- The interface that `w` must support is determined by the operations
performed on `w` in the template. In this example `w`'s type must support the
size, normalise, and swap member functions; copy construction;
comparison for inequality. ==The set of expressions that must be valid in order for the template to compile is the *implicit interface* that T must support.
- The calls to functions involving `w` such as `operator>` and `operator!=`
may involve instantiating templates. Such instantiation takes place during
compilation. Because instantiating function templates with different
template parameters leads to different functions being called, this is
known as ==compile-time polymorphism.==

An implicit interface is quite different to explicit interface. Consider the
start of doProcessing:
```cpp
template<typename T>
void doProcessing(T& w)
{
    if (w.size() > 10 && w! someNastWidget) { //this exprssn
    ...
```
The implicit interface for `w` appears to have these constraints:
- must offer a member function names size that returns an integral value
- must support `operator!=` that compares two objects of type T

Size need not return an integral type. All it needs is an object of some
type X such that there is an `operator>` that can be called with an object X
and an int. The `oeprator>` need not even take a parameter of type X,
because it could take a parameter of type Y as long as there is available
an implicit conversion between type X to object Y.

Similarly, there is no requirement that T support `operator!=`, because it
would be fine to  take one object of type X and one object of type Y, as
long as T can be converted to X and someNastyWidget's type can be
converted to Y.

This can seem confusing but it does not have to be. Implicit interfaces
are ==made up of a set of valid expressions.== It's easier to think about the
expression as a whole.  
`if (w.size() > 10 && w != someNastyWidget)...`  
An expression in an if statement must be a boolean expression. This
means that as long as "w.size() > 10 && w != someNastyWidget" yields
something compatible with a bool, the constraint will be met.

The implicit interfaces imposed on a template's parameters are just as
real as explicit interfaces imposed on a class's objects and both are
checked during compilation.

> [!abstract] Summary  
> - Both classes and templates support interfaces and polymorphism
> - For classes, interfaces are explicit and centered on function
> signatures. Polymorphism occurs at runtime through virtual functions.
> - For template parameters, interfaces are implicit and based on valid
> expressions. Polymorphism occurs during compilation through template
> instantiation and function overloading resolution.


### **Item 42:** Understand the two meanings of typename.
In a template declaration, class and typename is used interchangeably.
```cpp
// both mean the same
template<class T> class Widget;
template<typename T> class Widget;
```
However, class and typename don't always have the same meaning. The
meaning of class should already known. But we also have to know when
to use typename.

Few definitions:
- *dependent names* - Names in a template that are dependent on a
template parameter
- *nested dependent name* - When a dependent name is nested inside a
class

```cpp
template<typename C>
void print2nd(const C& container)
{
    C::const_iterator * x;
}
```
The 'const_iterator' name is nested inside the class C, which is a
template parameter. Thus it is a nested dependent name. In fact it is
actually a 'nested dependent typename' i.e. nested dependent name that
refers to a type.

However, nested dependent names can lead to parsing difficulties.
In theory, the `C::const_iterator * x;` could be saying "multiply a global
variable x with a static member in C called const_iterator". Until C is
known, we can't tell if it is a type or not. C++ has a rule to resolve
this ambiguity. ==It assumes that nested dependent names are not types==
unless told otherwise.

So what code that involves this would look like to be valid would be:
```cpp
template<typename C>
void print2nd(const C& container)
{
    if (container.size() >= 2) {
        typename C::const_iterator iter(container.begin());
        ...
```
Putting typename in front of C::const_iterator lets the compiler know
it is a type. This needs to be done every time you refer to a nested
dependent name in a template with one exception.

The exception is that typename must not precede nested dependent type
names in a list of base classes or as a base class identifier in a member
initialisation list. For example:
```cpp
template<typename T>
class Derived: public Base<T>::Nested { //typname not allowed
public:
    explicit Derived(int x)
    : Base<T>::Nested(x) //typename not allowed
    {
        typename Base<T>::Nested temp; // typename required
        ...
    }
};
```

Lets look at another example and how you might make it less tedious
to type out typename followed by a long verbose type.
```cpp
template<typename IterT>
void workWithIterator(IterT iter)
{
    typedef typename std::iterator_traits<IterT>::value_type value_type;
    value_type temp(*iter);
    ...
}
```

The `typedef typename` juxtaposition may be jarring initially but you'll get
used to using it and seeing it as the alternative is to type it all out in full
every time.

> [!abstract] Summary  
> - When declaring template parameters, `class` and `typename` are
> interchangeable.
> - Use `typename` to identify nested dependent type names, except in
> base class lists or as a base class identifier in a member
> initialisation list.


### **Item 43:** Know how to access names in templatised base classes.
Consider an application that can send message to several different
companies in either encrypted or cleartext form. If we have enough
information during compilation to determine which messages will go
to which companies, we can use a template based solution.
```cpp
class CompanyA {
public:
    ...
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string&* msg);
    ...
};
class CompanyB {
public:
    ...
    void sendCleartext(const std::string& msg);
    void sendEncrypted(const std::string&* msg);
    ...
};
...
class MsgInfo{...};

template<typename Company>
class MsgSender {
public:
    ...
    void sendClear(const MsgInfo& info)
    {
        std::string msg;
        //create msg from info
        Company c;
        c.sendCleartext(msg);
    }
    void sendSecret(const MsgInfo& info)
    {...}
};
```

Lets say we now want a derived class that adds logging before and after
we send a message:
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        //log before sending
        sendClear(info);
        //log after sending
    }
    ...
};
```
Note that message sending function is called `sendClearMsg` which is
different from the base class to side-step hiding inherited names.
However, the code won't compile as compilers will say `sendClear()` does
not exist. 

To explore why lets look at template specialisations quickly:
```cpp
class CompanyZ {
public:
    ...
    void sendEncrypted(const std::string& msg);
    ...
};

template<>
class MsgSender<CompanyZ> {
public:
    ...
    void sendSecret(const MsgInfo& info);
    {...}
};
```
The `template<>` syntax specifies that this is a specialised version of the
MsgSender template to be used when the template argument is `CompanyZ`.
This is known as a *total template specialisation*. Now when we look
back at the `LoggingMsgSender` template, `sendClear(info);` no longer
makes sense as it might not exist for certain companies like `CompanyZ`.

This is why the compilers generally refuse to look in templatised base
classes for inherited names. In a sense, when we cross from object
oriented C++ to template C++, inheritance stops working.

So how do we make it work again? Three ways:
- `this->`
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        //log before sending
        this->sendClear(info);
        //log after sending
    }
    ...
};
```

- `using MsgSender<Company>sendClear;`
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    using MsgSender<Company>::sendClear;
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        //log before sending
        sendClear(info);
        //log after sending
    }
    ...
};
```

- `MsgSender<Company>::sendClear(info);`
```cpp
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    ...
    void sendClearMsg(const MsgInfo& info)
    {
        //log before sending
        MsgSender<Company>::sendClear(info);
        //log after sending
    }
    ...
};
```

The third way is the least desirable way because if the function being
called is virtual, explicit qualification turns off the virtual binding
behaviour.

> [!abstract] Summary  
> In derived class templates, refer to names in base class templates
> via a `this->` prefix, via using declarations, or via an explicit base
> class qualification.

### **Item 44:** Factor parameter-independent code out of templates
Templates are a good way to save time and avoid code replication. Instead
of writing 20 similar classes, each with 15 member functions, you can
write one template. You can also do the same for function templates.

However, this can lead to *code bloat*. To counter this you need to do
*commonality and variability analysis*. This is something you already do
when writing non template code. You factor out common code if you
see that there are similar looking code in different functions.

The same needs to be done for templates. The catch is that in
non-template code, replication is explicit. Meaning you have to replicate
it by hand. In template code, replication is implicit. The code will be
replicated without additional input from you. You will have to train
yourself to bear this in mind.

```cpp
template<typename T, std::size_t n>
class SquareMatrix {
public:
    ...
    void inert();
};
...
SquareMatrix<double, 5> sm1;
sm1.invert();
SquareMatrix<double, 10> sm1;
sm1.invert();
```
In this example, two copies of invert will be instantiated. The functions
will be identical other than the constants 5 and 10.

One way to solve this is to put the parameterised version of invert into a
base class. SquareMatrixBase is templatised, but only in the type of
objects. This means all matrices holding a given type of object will use
the same `invert` function.
```cpp
template<typename T>
class SquareMatrixBase {
protected:
    ...
    void invert(std::size_t matrixSize);
    ...
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
private:
    using SquareMatrixBas<T>::invert;
public:
    ...
    void invert() {invert(n);}
};
```

However, we still have another issue. SquareMatrixBase::invert needs to
know what data to operate on. It's impractical to provide a pointer
to the data for all functions that SquareMatrix might have. One solution
is to have SquareMatrixBase store a pointer to the memory for the matrix
values. If it stores that, then it might as well also store the size:
```cpp
template<typename T>
class SquareMatrixBase {
protected:
    SquareMatrixBase(std::size_t n, T *pMem)
    : size(n), pData(pMem) {}
    void setDataPtr(T *ptr) { pData = ptr;}
    ...
private:
    std::size_t size;
    T *pData;
};
```
Now the base class can decide how to allocate the memory. The derived
class might choose to allocate the matrix inside the matrix object:
```cpp
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
public:
    SquareMatrix()
    : SquareMatrixBase<T>(n, data) {}
    ...
private:
    T data[n*n];
};
```
Objects such as these have no need for dynamic memory allocation but
the objects themselves could be very large. An alternative that puts data
on the heap could be:
```cpp
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {
public:
    SquareMatrix()
    : SquareMatrixBase<T>(n, 0),
      pData(new T[n*n])
      {this->setDataPtr(pData.get());}
    ...
private:
    boost::scoped_array<T> pData;
};
```

Regardless of where the data is stored, the key result from a code bloat
viewpoint is that many, perhaps all, of SquareMatrix's member functions
can be simple inline calls to base class versions. At the same time, 
SquareMatrix objects of different sizes are distinct types so there is no
way to accidentally pass an object of `SquareMatrix<double,5>` to a
function expecting `SquareMatrix<double,10>`.

However, there are some associated costs. The versions of invert with
the matrix sizes hardwired into them are likely to generate better code
than shared version where the size is passed as a function parameter. On
the other hand, one version of invert decreases the size of the executable
which might lead to improved locality of reference in the instruction
cache. Which is better? You will have to try both and measure.

This item discusses bloat due to non-type template parameters. However,
type parameters can cause bloat too. E.g. int and long can have same
representation on many platforms. Similarly, on most platforms all
pointer types have same binary representation, but each type pointer
can lead to bloat e.g. list<int*>, list<const int*>, 
list<SquareMatrix<long,3>\*>. This can be avoided using a strongly
typed pointer. This is what the STL does.

> [!abstract] Summary  
> - Templates generate multiple classes and multiple functions so any
> template code not dependent on a template parameter causes bloat.
> - Bloat due to non-type template parameters can often be eliminated
> by replacing template parameters with function parameters or class
> data members.
> - Bloat due to type parameters can be reduced by sharing
> implementations for instantiation types with identical binary
> representations.

### **Item 45:** Use member function templates to accept "all compatible types."
One of the things that raw pointers do well is support implicit conversions.
Derived class pointers implicitly convert into base class pointers.
Emulating such conversions in user-defined smart pointer classes is
tricky.

```cpp
class Top {...};
class Middle: public Top {...};
class Bottom: public Middle {...};
Top *pt1 = new Middle;
Top *pt2 = new Bottom;
const Top *pct = pt;
```
This example shows some of the implicit conversions that can take place
in a three-level hierarchy.

With smart pointers the code might look like this:
```cpp
tempalte<typename T>
class SmartPtr {
public:
    explicit SmartPtr(T *realPtr);
    ...
};
SmartPtr<Top> pt1 = SmartPtr<Middle>(new Middle);
SmartPtr<Top> pt2 = SmartPtr<Bottom>(new Bottom);
SmartPtr<const Top> pct = pt1;
```
However, this does not compile as there is no inherent relationship
among different instantiations of the same template. Compilers view
`SmartPtr<Middle>` and `SmawrtPtr<Top>` as completely different classes.

In principle, the number of constructors we would need could be unlimited
as the hierarchy can be indefinitely extended. It seems we need a
*constructor template* not a constructor function:
```cpp
template<typename  T>
class SmartPtr {
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other);
    ...
};
```
This says that for every type T and every type U, a `SmartPtr<T>` can be
create from a `SmartPtr<U>`, because `SmartPtr<T>` has a constructor
that takes a `Smartptr<U>` parameter. Constructors like this are sometimes
known as `generalised copy constructors`. This one is not marked explicit
as as type conversions between built-in pointer types are implicit and
require not cast, so it's reasonable for smart pointers to emulate that
behaviour.

However, this version now offers more than we want. We don't want to
be able to create a `SmartPtr<Top>` form a `SmartPtr<Bottom>` as that's
contrary to the meaning of public interface. We also don't want to be able
to create a `SmartPtr<int>` from a `SmartPtr<double>` as there is no
corresponding implicit conversion.

We can use the implementation of the constructor template to restrict the
conversions to those we want.
```cpp
template<typename T>
class SmartPtr {
public:
    template<typename U>
    SmartPtr(const Smartptr<U>& other)
    : heldPtr(ohter.get()) {...}
    T* get() const { return heldPtr;}
    ...
private:
    T *heldPtr;
};
```
Since we use the member initialisation list to to initialise `SmartPtr<T>`'s
data member of type T* with the pointer of type U*, this will only compile
if there is an implicit conversion from a U* pointer to a T*.

> [!abstract] Summary  
> - Use member function templates to generate functions that accept all
> compatible types.
> - If you declare member templates for generalised copy construction or
> generalised assignment, you'll still need to declare the normal copy
> constructor and copy assignment operator too.

### **Item 46:** Define non-member functions inside templates when type conversions are desired
Item 24 explains way only non-member functions are eligible for implicit
type conversions on all arguments.

```cpp
template<typename T>
class Rational {
public:
    Rational(const T& numberator = 0,
             const T& denominator = 1);
    const T numberator() const;
    const T denominator() const;
    ...
};
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs,
                            const Rational<T>& rhs)
{...}
```

As in item 24, we want to support mixed-mode arithmetic. So we want
the following code to compile
```cpp
Rational<int> oneHalf(1,2);
Rational<int> result = oneHalf * 2;  //error won't compile
```
From item 24, we know that `operator*` is being called with two
parameters. However, the compiler can't figure out which function to
instantiate. They need to figure out what T is but they can't.
```cpp
// is T int or Rational<T>?
Rational<int> result = operator*(onehalf, 2);
```

One parameter is a `Rational<T>` and the other is an int.
==Implicit type conversions are never considered during template argument deduction.==
We can relieve compilers of the challenge by taking advantage of the fact
that a friend declaration in a template class can refer to a specific
function. That means the class `Rational<T>` can declare `operator*` for
`Rational<T>` as a friend function.

```cpp
template<typename T>
class Rational {
public:
    ...
friend
    const Rational operator*(const Rational& lhs,
                             const Rational& rhs);
};
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs,
                             const Rational<T>& rhs)
{...}
```
This wont' still link just yet. But first a quick note about the syntax.
==Inside a class template, the name of the template can be used as shorthand for the template and it's parameters==.
So inside `Rational<T>` we can just write `Rational` instead of
`Rational<T>`.

Back to the linking problem. If we declare a function ourselves, we're
also responsible for defining that function. In this case we never provide
one and that's why the linker's can't find one. The simplest solution is
to merge the body of `operator*` in the declaration. This will compile
and link fine.

The other approach worth mentioning is the "have a friend call a helper"
method.
```cpp
template<typename T> class Rational; //declare
template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs
                             const Rational<T>& rhs);

template<typename T>
class Rational {
public:
    ...
friend
    const Rational<T> operator*(const Rational<T>& lhs,
                                const Rational<T>& rhs)
    { return doMultiply(lhs, rhs); } 
    ...
};
```

> [!abstract] Summary  
> When writing a class template that offers functions related to the
> template that support ==implicit conversions== on all parameters, define
> those functions as friends inside the class template.

### **Item 47:** Use traits classes for information about types.
There are 5 kinds of iterator categories:
- *Input iterators*. Only move forward. Only move one step at a time. Only
read what they're pointed to and only once.
- *Output iterators*. Only move forward. Only one step at a time. Only
write where they're pointed and only once.
- *Forward iterators*. All of above plus can read or write more than once.
- *Bidirectional iterators*. Similar to forward iterators but also backwards.
e.g. `std::list`
- *Random access iterators*. Adds 'iterator arithmetic' to bidirectional
iterator i.e. jump forward or backward an arbitrary distance in constant
time.

For each iterator category, C++ has a "tag struct" in the STL to identify
it.
```cpp
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator: public input_iterator_tag {};
struct bidrectional_iterator_tag: public forward_iterator {};
struct random_access_iterator_tag: public bidirectional_iterator_tag {};
```

To implement `std::advance` you can use 'iterator arithmetic' to move
a random access iterator a specific distance or you can increment the
iterator a certain amount of times in a loop. This requires being able to
tell what kind of iterator you are working on.
==This is what traits let you do. They allow you to get information about a type during compilation.==

Traits aren't a keyword. They are a technique and convention followed by
C++ programmers. One demand on the technique is that it has to work
for built-in types as well as user defined types. This means you can't
nest information about a type as you can't do this with pointers.

The standard technique is to put it into a template and one or more
specialisations for that template.
```cpp
tempalte<typename iterT>
struct iterator_traits;
```

The way `iterator_traits` works is that for any iterator type (iterT), a
typedef named iterator_category is declared in the struct of
`iterator_traits<iterT>`. This will be used to identify the iterator
category of `iterT`.

```cpp
template<...>
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        ...
    };
    ...
};
```

iterator_traits just parrots back the iterator class's nest typedef:
```cpp
template<typename IterT>
struct iterator_traits {
    typedef typename IterT::iterator_catebory iterator_category;
    ...
};
```

This works well for user-defined types, but not at all for pointers. The
second part of implementing iterator_traits is to create a `partial template specialisation`
for pointer types. Pointers act as random access iterators so that's what
we'll use.
```cpp
tempalte<typename T>
struct iterator_traits<T*>
{
    typedef random_access_iterator_tag iterator_category;
}
```

Given `iterator_traits` we can now refine our pseudocode for advance:

```cpp
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (typeid(typename std::iterator_traits<IterT>::iterator_category) ==
        typeid(std::random_access_iterator_tag))
        ...
}
```

There is one more problem. We know the type during compilation but we
are evaluating it at runtime in the if statement. What we want is
conditional construct for types that is evaluated during compilation. We
can get that behaviour with overloading.

When you overload a function, compilers will pick the best overload based
on the arguments you're passing. A compile time conditional for types. To
get `advance` to behave the way we want, we have to create multiple
versions of an overloaded function declaring each to take a different type
of iterator_category object.

```cpp
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d, 
             std::random_access_iterator_tag)
{
    iter += d;
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d,
             std::bidirectional_iterator_tag)
{
    if (d >= 0) { while (d--) ++iter; }
    else { while (d++) --iter;}
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, DistT d,
             std::input_iterator_tag)
{
    if (d < 0) {
        throw std::out_of_range("Negative distance");
    }
    while (d--) ++iter;
}
```

`forward_iterator_tag` inherits from `input_iterator_tag` so the version of
doAdvance for `input_iterator_tag` will also work for it.

Now, given the overloads for `doAdvance`, all `advance` needs to do is call
them.
```cpp
template<typename IterT, typename DistT>
void advance(IterT iter, DistT d)
{
    doAdvance(iter, d,
              typename std::iterator_traits<IterT>::iterator_category());
}
```

**How to use traits class**
- Create a set of overloaded "worker" functions or function templates
that differ in a traits parameter. Implement each in accord with the traits
info passed.
- Create a "master" function or function template that calls the workers
passing information provided by a traits class.

> [!abstract] Summary  
> - Traits classes make information about types available during
> compilation. They're implemented using templates and template
> specialisation.
> - In conjunction with overloading, traits make it possible to perform
> compile-time `if...else` tests on types.

### **Item 48:** Be aware of template metaprogramming.
Template metaprogramming (TMP) is the process of writing template based
C++ programs that execute during compilation. It is a program that
executes inside the C++ compiler. That is bizarre.

TMP has two great strengths:
- Makes some things easy that would otherwise be hard or impossible.
- Can shift work form runtime to compile time. Some consequences of this are:
    - Some errors usually detected at runtime can then be detected at compile time.
    - Programs can be more efficient.
    - Compilation takes longer.

Consider the typeid based approach of advance from the previous item:
```cpp
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    if (typeid(typename std::iterator_traits<IterT>::iterator_category) ==
        typeid(std::random_access_iterator_tag))
    {
        iter += d;
    }
    else
    {
        if (d >= 0) {while (d--) ++iter;}
        else {while (d++) --iter};
    }
}
```

One of the strengths of TMP was that some things that are hard are now
easy with TMP. This is one example:
```cpp
std::list<int>::iterator iter;
...
advance(iter, 10);
```
This will *NOT* compile. Why? Because `iter += d` does not work on list
iterators which are bidirectional. We know it will never go in the `if` block
but compilers are obliged to make sure all source code is valid even if not
executed.

TMP is turing complete. For a quick glimpse of what is possible lets look
at the "hello world" of TMP, computing a factorial.
```cpp
//general case. Factorial n = n times factorial n - 1
template<unsigned n>
struct Factorial {
    enum { value = n * Factorial<n-1>::value};
};

//special case. factorial 0 = 1
template<>
struct Factorial<0> {
    enum { value = 1};
}
```

This uses the 'enum hack' to store the value. Now to use factorial, you
can do so like this:
```cpp
int main()
{
    std::cout << Factorial<5>::value; //prints 120
    std::cout << Factorial<10>::value; //prints 3628800
}
```

Here are 3 examples to describe the power of TMP:
- **Ensuring dimensional unit correctness.** in scientific and engineering
applications, it's essential to combine units(e.g mass, distance, time
etc) correctly. Using TMP, it's possible to ensure (during compilation)
that all dimensional unit combinations in a program are correct no matter
how complex the calculations. One interesting aspect of this use of TMP
is that fractional dimensional exponents can be supported. This requires
fractions to be reduced *during compilation* so that compilers can confirm
that "time 1/2" is the same as "time 4/8".
- **Optimising matrix operations.** Item 21 talked about how some functions
must return new objects.
```cpp
typedef SquareMatrix<double, 10000> BigMatrix;
Bigmatrix m1, m2, m3, m4, m5; //create matrices
...                           //give them values
BIgMatrix result = m1 * m2 * m3 * m4 * m5; //compute product
```
Calculating the normal way requires the creation of 4 temporary matrices.
Furthermore, the independent multiplications generate a sequence of four
loops over the matrix elements. Using advance template technology related
to TMP called *expression templates*, it's possible to eliminate the
temporaries and merge the loops, all without changing the syntax of the
client code above.
- **Generating custom design pattern implementations.** Design patterns like
Strategy, Observer, Visitor, etc can be implemented in many ways. Using
TMP based technology called *policy-based design*, it's possible to create
templates representing independent design choices ("policies") that can
be combined in arbitrary ways to yield pattern implementations with
custom behaviour. For example, this technique has been used for a few
templates implementing smart pointer behavioral policies to generate
(during compilation) any of hundreds of different smart pointer types.
This is a basis for what is known as generative programming.

> [!abstract] Summary  
> - TMP can shift work from runtime to compile time, enabling earlier
> detection and higher runtime performance.
> - TMP can generate custom code based on combination of policy
> choices, and also be used to avoid generating code inappropriate for
> particular types.

## Ch8: Customising `new` and `delete`  

### **Item 49:** Understand the behaviour of the new-handler

```cpp
namespace std {
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
```

`set_new_handler`'s parameter is a pointer to the function operator `new`
should call if it can't allocate the requested memory. The return value of
`set_new_handler` is a pointer to the previous function that would have
been called.

You can use `set_new_handler` like so:
```cpp
void outOfMem()
{
    std::cerr << "Unable to satisfy requeset for memory\n";
    std::abort();
}

int main()
{
    std::set_new_handler(outOfMem);
    int* pBigDataArray = new int[100000000L];
    ...
}
```

When operator `new` is unable to fulfill a memory request, it calls the
new_handler repeatedly until it *can* find enough memory. A new_handler
must do one of the following:
- **Make more memory available**
- **Install a different new-handler**
- **Deinstall the new-handler**
    - Pass `nullptr` to `set_new_handler` which will cause operator `new` to throw an exception next time memory allocation is unsuccessful.
- **Throw an exception**
- **Not return**
    - Usually by aborting

To implement 'class-specific new-handler' like behaviour:
```cpp
class Widget {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler currentHandler;
};

std::new_handler Widget::set_new_handler(std::new_handler p) throw()
{
    std::new_handler oldHandler = currentHandler;
    currentHandler = p
    return oldHandler;
}
```

The `Widget`s operator `new` will do the following:
- Call the standard `set_new_handler` with Widget's error handling
function. This installs widgets new_handler as the global new_handler.
- Call global operator `new` to perform memory allocation.
- If allocation fails, call Widgets new_handler. If it still can't allocate
the memory, restore the original new_handler, then propagate exception.
You can use a RAII object for this.
- if allocation succeeds, the destructor for the object managing the global
new_handler restores the global new_handler to what it was.

e.g.
```cpp
class NewHandlerHolder {
public:
    explicit NewHandlerHolder(std::new_handler nh)
    : handler(nh) {}
    ~NewHandlerHolder()
    {std::set_new_handler(handler);}
private:
    std::new_handler handler;
    NewHandlerHolder(const NewHandlerHolder&);
    NewHandlerHolder& operator=(const NewHandlerHolder&)''
}

// WIdget's operator new
void* Widget::operator new(std::size_t size) throw(std::bad_alloc)
{
    NewHandlerHolder
      h(std::set_new_handler(currentHandler));
    return ::operator new(size);
}
```

Client's of Widget would use it like so:
```cpp
void outOfMem();

//set outOfMem as widget's new-handling func
Widget::set_new_handler(outOfMem);

// calls outOfMem if allocation fails
Widget* pw1 = new Widget;
// calls global new-handling function if allocation fails
std::string* ps = new std::string;

// set new-handling func to nothing
Widget::set_new_handler(0);

//if mem. alloc fails, throw exception immediately
Widget* pw2 = new Widget;
```

The above NewHandlerHolder can be adapted very easily to be used as a
"mixin-style" base class using templates. Then Widget could just inherit
from that class:
```cpp
template<typename T>
class NewHandlerHolderSupport {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler currentHandler;
};
...
// implement functions from interface
...
class Widget: public NewHandlerSupport<Widget> {
...
};
```

Widget inheriting from a templatised base class that takes `Widget` as a
type parameter might look strange but it is a useful technique. It even
has a name. ==Curiously recurring template pattern (CRTP)==.

Until 1993, C++ required that `new` return null when it was unable to allocate
the requested memory. Now it throws `bad_alloc`. To support the old style
new C++ offers a nothrow form of `new`.
```cpp
Widget* pw2 = new(std::nothrow) Widget;
```
Unfortunately, this is not very useful as it is only nothrow for the
allocation part. Constructing the object can still throw.

> [!abstract] Summary  
> - set_new_handler allows you to specify a function to be called when
> memory allocation requests cannot be satisfied.
> - Nothrow new is of limited utility, because it applies only to memory
> allocation; associated constructor calls may still throw exceptions.

### **Item 50:** Understand when it makes sense to replace new and delete
Three most common reasons to replace compiler version of operator `new`
or `delete`.
- **To detect usage errors**
    - A custom operator new can keep list of allocated addresses and operator delete can remove from the list. This can easily detect usage errors such as failure to delete.
    - Custom operator new can allocate blocks alongside signatures. Then operator delete can check if the signatures are intact. This can detect an overrun or underrun.
- **To improve efficiency**
    - Compiler versions of new and delete are general purpose. They have to work well with programs that run for less than a second as well as something like web servers. By using a more specialised version, it can often be easy to increase the performance of your program.
- **To collect usage statistics**
    - Before optimising, you can gather data on how your program uses and allocates memory. What is the distribution of allocated block sizes, their lifetimes, pattern the blocks are allocated and de-allocated (FIFO or LIFO or random), maximum amount of allocated memory used at a given time etc.

Simple (with problems) operator `new` that facilitates under and overruns.
```cpp
static const int signature = 0xDEADBEEF;
typedef unsigned char Byte;

void* operator new(std::size_t size) throw(std::bad_alloc)
{
    using namespace std;

    // increase size of request to fit signature
    size_t realSize = size + 2 * sizeof(int);
    // call malloc to get memory
    void* pMem = malloc(realSize);
    if (!pMem) throw bad_alloc();
    //write signature into first and last parts of memory
    *(static_cast<int*>(pMem)) = signature;
    *(reinterpre_cast<int*>(static_cast<Byte*>(pMem) + realSize-sizeof(int))) = signature;
    // return pointer to the memory just past first sig
    return static_cast<Byte*>(pMem) + sizeof(int);
}
```

Most of the shortcomings of this have to do with failure to adhere to
the C++ conventions for operator `new`. However, one more subtle
issue is **alignment**.

Many computer architectures require that data of particular types be
placed in memory at particular kinds of addresses. E.g. a requirement
might be that pointers occur at addresses that are a multiple of 4
(i.e. be *four-byte aligned*) or that doubles must occur at addresses
that are multiples of 8 (*eight-byte aligned*). Failure could result in
hardware exceptions for some architectures. More forgiving ones
might just result in degraded performance.

Details like this make writing professional-quality memory managers
difficult. Don't try it unless you have to. An option is to use open
source memory managers. One example is the Pool library from Boost.
It optimises for allocation of a large number of small objects.

We can now add more bullet points to why we might want to use custom
version of `new` and `delete`
- To detect usage errors
- To collect statistics about the use of dynamically allocated memory
- To increase the speed of allocation and deallocation.
- To reduce space overhead of default memory management.
- To compensate for suboptimal alignment in default allocator.
- To cluster related objects near one another.
- To obtain unconventional behaviour - e.g. Deallocate blocks in shared memory via a C API.

> [!abstract] Summary  
> There are many valid reasons for writing custom versions of `new`
> and `delete`. Some include improving performance, debugging heap
> usage errors, and collecting heap usage information.

### **Item 51:** Adhere to convention when writing `new` and `delete`

Implementing a conformant operator `new` requires:
- have right return value
- call new-handling function when insufficient memory available
- cope with requests for no memory
- avoid hiding the "normal" `new`

Here is a simple implementation:
```cpp
void* operator new(std::size_t size) throw (std::bad_alloc)
{
    using namespace std;
    if (size == 0) {
        size = 1;
    }

    while (true)
    {
        //attempt to allocate size bytes
        if (/*allocation was successful*/)
            return /*pointer to memory*/;

        // allocation unsuccessful. Get current new_handler
        new_handler globalHandler = set_new_handler(0);

        if (globaHandler)(*globalHandler)();
        else throw std::bad_alloc();
    }
}
```

For operator `new[]`, remember you can't figure out how many objects will
be in the array or even how big an object is. This is because  a base
class's `new[]` might be used, through inheritance, to allocate memory for
an array of derived class objects. All you can do is allocate a chunk of
memory. All you need to remember is to make sure it is safe to delete a
null pointer.

`size_t` values passed to operator `delete` may be incorrect if the object
being deleted was derived from a base class lacking a virtual destructor.
Make sure you're base classes have virtual destructors.

> [!abstract] Summary  
> - Operator `new` should contain an infinite loop trying to allocate
> memory, should call new-handler if it can't satisfy a memory request,
> and should handle requests for zero bytes. Class specific versions
> should handle requests for larger blocks than expected.
> - Operator `delete` should do nothing if passed a pointer that is null.
> Class specific versions should handle blocks that are larger than
> expected.


### **Item 52:** Write placement `delete` if you write placement `new`

For a line such as this:
```cpp
Widget *pW = new Widget;
```
Two function calls are made, one to operator new to allocate memory and
another to the constructor of Widget. If the first call succeeds but the
second one does not, the memory allocation has to be undone. However,
client code is unable to do this as `*pW` was never assigned.

The C++ runtime must handle this responsibility. With normal forms of  `new`
and `delete` it is not a problem. However, if you have a class specific one
with extra parameters it is known as a ==placement== version of `new`.

There is an 'original' version of placement new in the STL:
```cpp
void operator new(std::size_t, void* pMemory) throw();
```
Sometimes when people talk about 'placement new' they are talking about
this specific original version. Be mindful what one they are talking about.

Now, if you use a placement version of new and the constructor call fails,
the runtime system will look for a version of new with the same number
of parameters and types. If it finds it, it calls it. If it does not, then no
delete will be called.

To eliminate memory leaks, you need to specify a matching version of
placement `delete`.
```cpp
class Widget {
public:
    ...
    // class specific placement new
    static void* operator new(std::size_t size, std::ostream& logStream)
      throw(std::bad_alloc);
    // class specific delete - when no exception throw
    static void operator delete(void* pMemory, std::size_t size)
      throw();
    // class specific placement delete - when exceptions thrown
    static void operator delete(void* pMemory, std::ostream& logStream)
      throw();
};
```

One thing to bear in mind that when you declare class specific `new` and
`delete`, this will hide the global versions. Be sure to make them
available in addition to any custom operator `new` forms you create. This
can be done by creating a base class containing the normal global
versions and inheriting from it.

> [!abstract] Summary  
> - When you write a placement version of operator `new`, be sure to
> write the corresponding placement version of operator `delete`. If you
> don't, there may be subtle, intermittent memory leaks.
> - When you declare placement versions of `new` and `delete`, be sure not
> to unintentionally hide the normal versions of those functions.


## Ch9: Miscellany  

### **Item 53:** Pay attention to compiler warnings
Compiler warnings should not be ignored in C++ with the thought that "if
it was really a big problem, it would be an error". Here is one example:
```cpp
class B {
public:
    virtual void f() const;
};

class D: public B {
public:
    virtual void f();
};
```
The intent is for `D::f` to redefine `B::f` but the mistake is that the version
in D is not const. The compiler might warn you by saying something like
"Warning: D::f() hides virtual B::f()". It is a mistake to think,
"of course it does. That's what i'm trying to do". Anyhow, if this is
ignored, it will almost certainly lead to erroneous behaviour which will
require a lot of debugging to later discover.

After you gain experience with warning messages from a particular
compiler, you'll learn to understand the different messages. At that point
you can then opt to ignore certain classes if warnings, although it is
better practice to strive for warning free code.

Relying on warnings is also not a good idea as there will be differences
across different compilers. You should strive to write warning free code
without relying on the compilers warning to catch mistakes. A different
compiler might not provide the warning message that you have become
reliant on getting.

> [!abstract] Summary  
> - Take compiler warnings seriously and strive to compile warning-free
> at the maximum warning level supported by your compiler.
> - Don't become dependent on compiler warnings because different
> compilers will warn you about different things. Moving compilers may
> eliminate a warning that you have become reliant on.


### **Item 54:** Familiarise yourself with the standard library, including TR1
Initial standard for C++ was ratified in 1998. in 2003, a minor 'bug-fix'
update was issued. The standardisation committee continued it's work,
however, and a "Version 2.0" C++ standard was adopted in 2011.

C++11 includes a number of interesting new language features, but most
come in the form of additions to the standard library. Before we look at
C++11, lets look at what was included in the standard library by C++98:
- **The standard template library (STL)**, including containers;
iterators; algorithms; function objects; and various container and function
object adapters.
- **Iostreams**, including support for user-defined buffering, internalised IO
and predefined objects `cin`, `cout`, `cerr` and `clog`.
- **Support for internationalisation**. Multiple active locales. `wchar_t`.
`wstring`. 
- **Support for numerical processing**, including templates for complex
numbers and arrays of pure values.
- **An exception hierarchy**
- **C89's standard library**

Technical Report 1 (TR1) specifies 14 new components. All are in the std
namespace (used to be nested within std in the tr1 namespace).
- **Smart pointers**
- **tr1::function**. Makes it possible to represent *any* callable entity.
- **tr1::bind**
- **Hash tables**
- **Regular expressions**
- **Tuples**
- **tr1::array**
- **tr1::mem_fn**. Syntactically uniform way of adapting member function pointers
- **tr1::reference_wrapper**. facility to make references act a bit more like objects.
- **Random number generation**
- **Mathematical special functions**, including Laguerre polynomial, Bessel functions etc.
- **C99 compatibility extensions**, collection of functions and templates
designed to bring many new C99 features to C++.
- **Type traits**
- **tr1::result_of**, a template to deduce return type of function calls.

> [!abstract] Summary  
> - The primary C++ library functionality consists of the STL, iostreams,
> and locales.
> - TR1 added support for smart_pointers, generalised function pointers
> , hash-based containers, regular expressions and 10 more components.


### **Item 55:** Familiarise yourself with Boost.
Look to boost for the following:
- High quality, open source, platform and compiler-independent libraries
- Community of ambitious, talented C++ developers working on
state-of-the-art library design and implementation
- Glimpse of what C++ might look like in the future

Boost was founded by committee members and there is strong overlap between Boost and committee memberships.

It's process for accepting libraries is based on peer review. This keeps
poorly written libraries out of Boost, but also helps to educate library
authors in the considerations that go into design, implementation, and
documentation of industrial-strength cross-platform libraries.

Boost covers a large cornucopia of topics. These categories include:
- **String and text manipulation**
- **Containers**
- **Function objects and higher-order programming**
- **Generic programming**
- **Template metaprogramming**
- **Math and numerics**
- **Correctness and testing**
- **Data structures**
- **Inter-language support**, including a library to allow seamless
interoperability between C++ and python.
- **Memory**, including pool library for high-performance fixed-size allocators.
- **Miscellaneous**: CRC checking, date and time manipulations, traversing
file systems.

> [!abstract] Summary  
> - Boost is a community and web site for the development of free,
> open source, peer-reviewed C++ libraries. Boost plays an influential
> role in C++ standardisation.
> - Boost offers implementations of many TR1 components, but it also
> offers many other libraries.

