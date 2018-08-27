---
title: Qt Reversing
categories:
  - Reverse Engineering
tags:
  - Assembly
  - C++
  - Qt
  - x86
---

I came across an application the other day that I was interested in taking a peek at under the hood. The first thing I did was to search through all the strings to see if there was anything interesting. Two things jumped out at me. The first was a lot of strings that looked like this: `cClass1`, `cClass2`, `cClass3`. Alright seems like it could be something written in C++. And then I also came across this string, `qt-win-commercial-3.2.3` Alright so this information gives me a lot to start with. It is for sure C++ code and what's more I now know what GUI toolkit it's using. So first thing I usually do when looking at a C++ applications is to try to find the RTTI information since it makes identifying code much easier. I started by checking references to the class name strings I had found. Unfortunately I didn't find any RTTI information. What I did find however were lots of functions that looked like this 

```
004E574B   . B8 1C217700    MOV EAX,app.0077211C     ;  ASCII "cClass1"
004E5750   . C3             RETN
```

This immediately jumps out at me as some sort of `Class.getName()` type thing. If I had of known anything about Qt I would have known right away that Qt doesn't use normal C++ RTTI information, but they use their own custom stuff.

Anyway this led me down the path of finding out more about Qt. Since Qt is open source it made this whole thing a lot easier. The way Qt does RTTI information is with a layer they build on top of normal C++ classes. They use custom macros that are put into the classes and then use their meta object compiler to include all the extra functions of the class to do things like dynamic typing. The easiest way to see this is with an example.

```c++
class MyClass : public QObject {
    Q_OBJECT
public: 
    MyClass( QObject * parent=0, const char * name=0 );
    MyClass();
signals:
    void mySignal();
public slots:
    void mySlot();
};
```

Here's the output from moc (meta object compiler)

```c++
#if !defined(Q_MOC_OUTPUT_REVISION)
#define Q_MOC_OUTPUT_REVISION 2
#elif Q_MOC_OUTPUT_REVISION != 2
#error "Moc format conflict - please regenerate all moc files"
#endif

#include "./MyClass.h"
#include 


const char *MyClass::className() const
{
    return "MyClass";
}

QMetaObject *MyClass::metaObj = 0;


#if Qt_VERSION >= 200
static QMetaObjectInit init_MyClass(&MyClass::staticMetaObject);
#endif

void MyClass::initMetaObject()
{
    if ( metaObj )
        return;
    if ( strcmp(QWidget::className(), "QObject") != 0 )
        badSuperclassWarning("MyClass","QObject");
    
#if Qt_VERSION >= 200
    staticMetaObject();
}

void MyClass::staticMetaObject()
{
    if ( metaObj )
        return;
    QObject::staticMetaObject();
#else

    QObject::initMetaObject();
#endif

    m1_t0 v1_0 = &AccountTable::mySlot;
    m1_t1 v1_1 = &AccountTable::mySignal;
    QMetaData *slot_tbl = new QMetaData[1];
    slot_tbl[0].name = "mySlot()";
    slot_tbl[0].ptr = *((QMember*)&v1_0);
    QMetaData *signal_tbl = new QMetaData[1];
    signal_tbl[0].name = "mySignal()";
    signal_tbl[0].ptr = *((QMember*)&v1_1);
    metaObj = new QMetaObject( "MyClass", "QObject", slot_tbl, 1, signal_tbl, 1 );
}
```

So now things get interesting. We can clearly see that for every Qt class that is written, moc will generate a `className()` function as well as generating a function called `staticMetaObject` that hooks up the class hierarchy and the singals and slots that Qt uses. This is great news for reversing because it means not only do we have class names and class hierarchies, but we also have function names for slots and signals.

So after finding all of this out I set about writing a script to automatically label all the class vtables and label all moc generated functions. The way I did it was like this. I searched for all sequence of bytes that looked like this `B8 * * * * C3` So basically find all of the `className()` methods. I narrowed down any false positives by making sure that the string that was referenced started with a lower case c since all the class names in this particular application started with a c. After I found the `className` method the script followed the reference to it to label the vtable for the class. Once the vtable is found since moc always generates the same functions in the same order it was a simple matter of labeling the first handful of function pointers in the vtable for the class to the auto generated moc methods.

The final key thing that is helpful to know is that once you have the `staticMetaObject` for a Qt class found you now know what the parent class is based on what `staticMetaObject` method is called. You also know the definition of the `signal_tbl` and the `slot_tbl`. If you trace references to these two tables you can find the references to the actual implementations of these methods and label those as well.
