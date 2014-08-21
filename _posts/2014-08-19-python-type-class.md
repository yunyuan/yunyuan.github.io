---
layout: post
category : Python
title: Python中的 type() 和 __class__
description : 最近在公司内部的问答系统上有同事问了一个问题：`Python`中的`type()`和`__class__`有什么差别？
tags : [Python]
---


最近在公司内部的问答系统上有同事问了一个问题：`Python`中的`type()`和`__class__`有什么差别？


    >>> class Foo(object):
        pass

    >>> class Bar(object):
        pass

    >>> class Brion(object):
        pass

    >>> class ASML(object):
        __class__ = Foo


        
    >>> b = Bar()
    >>> a = ASML()


#### Case 1

    >>> b.__class__, type(b)
    (<class '__main__.Bar'>, <class '__main__.Bar'>)
    >>>
    >>> b.__class__ = Foo
    >>>
    >>> b.__class__, type(b)
    (<class '__main__.Foo'>, <class '__main__.Foo'>)


#### Case 2

    >>> a.__class__, type(a)
    (<class '__main__.Foo'>, <class '__main__.ASML'>)
    >>>
    >>> a.__class__ = Brion
    >>>
    >>> a.__class__, type(a)
    (<class '__main__.Brion'>, <class '__main__.ASML'>)


大家看出`Case 1`和`Case 2`的差别了吧。问题来了：  
  1. `type(obj)`到底做了些什么事情？  
  2. 为什么`a`在改变`__class__`后，`type(a)`还是`ASML`呢？

为了解决这些问题，我们需要深入`Python`源代码。以下源代码来自`Python 2.7.8`。

Python Object
---
`Python`中一切皆对象。所有对象的数据结构都以一个`PyObject_HEAD`开头。 

**object.h**

    typedef struct _object {
        PyObject_HEAD
    } PyObject;

    /* PyObject_HEAD defines the initial segment of every PyObject. */
    #define PyObject_HEAD                   \
        _PyObject_HEAD_EXTRA                \
        Py_ssize_t ob_refcnt;               \
        struct _typeobject *ob_type;

这里的`ob_refcnt`是用来做引用计数的，而`ob_type`则是对象所对应的`type`对象。

`type`对象包含了很多关于对象的元信息：类型名字(`tp_name`)，创建该类型对象时分配内存空间大小的信息(`tp_basicsize`和`tp_itemsize`)，一些操作信息(`tp_call`, `tp_new`等)，还有其他如`__mro__`(`tp_mro`), `__bases__`(`tp_bases`)等。 

**object.h**

    typedef struct _typeobject {
        PyObject_VAR_HEAD
        const char *tp_name; /* For printing, in format "<module>.<name>" */
        Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

        ...

        /* More standard operations (here for binary compatibility) */
        ternaryfunc tp_call;
        ...

        /* Attribute descriptor and subclassing stuff */
        PyObject *tp_dict;
        newfunc tp_new;
        PyObject *tp_bases;
        PyObject *tp_mro; /* method resolution order */
        PyObject *tp_subclasses;
        ...

        ...

    } PyTypeObject;


对于对象`f = Foo()`来说，它的`ob_type`就是`Foo`, 而`Foo`的`ob_type`则是`type`。

`type(obj)`到底做了些什么事情
---
上面提到的每个对象都有的`ob_type`其实就是`type(obj)`返回的对象。

我们看下运行`type(obj)`时调用到的一系列函数。

**abstract.c**

    PyObject *
    PyObject_Call(PyObject *func, PyObject *arg, PyObject *kw)
    {
        ternaryfunc call;

        if ((call = func->ob_type->tp_call) != NULL) {
            PyObject *result;
            if (Py_EnterRecursiveCall(" while calling a Python object"))
                return NULL;
            result = (*call)(func, arg, kw);
            ...
            return result;
        }
        ...
        return NULL;
    }

这里的`func`是`obj`的`ob_type`，以`f = Foo()`为例的话就是`Foo`。那`func->ob_type`当然就是`type`了。

**typeobject.c**

    PyTypeObject PyType_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "type",                                     /* tp_name */
        ...
        (ternaryfunc)type_call,                     /* tp_call */
        ...
        type_new,                                   /* tp_new */
        ...
    }



从上面`PyType_Type`的定义可以看到`type`的`tp_call`就是`type_call`函数：

**typeobject.c**

    static PyObject *
    type_call(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
        PyObject *obj;

        if (type->tp_new == NULL) {
            PyErr_Format(PyExc_TypeError,
                         "cannot create '%.100s' instances",
                         type->tp_name);
            return NULL;
        }

        obj = type->tp_new(type, args, kwds);
        if (obj != NULL) {
            /* Ugly exception: when the call was type(something),
               don't call tp_init on the result. */
            if (type == &PyType_Type &&
                PyTuple_Check(args) && PyTuple_GET_SIZE(args) == 1 &&
                (kwds == NULL ||
                 (PyDict_Check(kwds) && PyDict_Size(kwds) == 0)))
                return obj;                         // Yun: type(obj) returns from here
            /* If the returned object is not an instance of type,
               it won't be initialized. */
            if (!PyType_IsSubtype(obj->ob_type, type))
                return obj;
            type = obj->ob_type;
            if (PyType_HasFeature(type, Py_TPFLAGS_HAVE_CLASS) &&
                type->tp_init != NULL &&
                type->tp_init(obj, args, kwds) < 0) {
                Py_DECREF(obj);
                obj = NULL;
            }
        }
        return obj;                              // Yun: type(cls, bases, dict) returns from here


`type_call`中先调用`tp_new`指向的函数(`type_new`)，然后再做分支。对`type(obj)`调用来说就是直接返回`tp_new`得到的对象。而对`type(cls, bases, dict)`来说还会调用`tp_init`指向的函数，这在自定义`metaclass`时会用到。  

**typeobject.c**

    static PyObject *
    type_new(PyTypeObject *metatype, PyObject *args, PyObject *kwds)
    {
        ...

        /* Special case: type(x) should return x->ob_type */
        {
            const Py_ssize_t nargs = PyTuple_GET_SIZE(args);
            const Py_ssize_t nkwds = kwds == NULL ? 0 : PyDict_Size(kwds);

            if (PyType_CheckExact(metatype) && nargs == 1 && nkwds == 0) {
                PyObject *x = PyTuple_GET_ITEM(args, 0);
                Py_INCREF(Py_TYPE(x));
                return (PyObject *) Py_TYPE(x);
            }

            ...
        }

        ...
    }


**object.h**

    #define Py_TYPE(ob)             (((PyObject*)(ob))->ob_type)


可见`type(obj)`其实就是返回对象的`ob_type`。

为什么`a`在改变`__class__`后，`type(a)`还是`ASML`呢
---
要回答这个问题，我们要先回顾下通过`obj.xxx`查找对象的 attribute 时的搜索顺序：  
 1. type对象及其基类的`__dict__`。如果是 data descriptor，返回这个 data descriptor的 `__get__` 结果  
 2. obj的`__dict__`  
 3. 第一步中找到的如果是 non-data descriptor， 返回这个 non-data descriptor的 `__get__` 结果  
 4. type对象中的`__dict__`，也就是直接返回第一步中找到的对象  

`obj.__class__`就是一个 attribute 查找。

**typeobject.c**

    static PyGetSetDef object_getsets[] = {
        {"__class__", object_get_class, object_set_class,
         PyDoc_STR("the object's class")},
        {0}
    };

    PyTypeObject PyBaseObject_Type = {
        PyVarObject_HEAD_INIT(&PyType_Type, 0)
        "object",                                   /* tp_name */
        ...
        PyObject_GenericGetAttr,                    /* tp_getattro */
        PyObject_GenericSetAttr,                    /* tp_setattro */
        ...
        object_getsets,                             /* tp_getset */
        ...
    };


来看下`f = Foo(); f.__class__;`中的函数调用链：

**object.c**

    PyObject *
    PyObject_GetAttr(PyObject *v, PyObject *name)
    {
        PyTypeObject *tp = Py_TYPE(v);

        ...

        if (tp->tp_getattro != NULL)
            return (*tp->tp_getattro)(v, name);

        ...
        return NULL;
    }

这里的`v`就是`f`, 而`tp`就是`Foo`, `tp->tp_getattro`就是`PyObject_GenericGetAttr`函数。

**object.c**

    PyObject *
    PyObject_GenericGetAttr(PyObject *obj, PyObject *name)
    {
        return _PyObject_GenericGetAttrWithDict(obj, name, NULL);
    }

    PyObject *
    _PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict)
    {
        PyTypeObject *tp = Py_TYPE(obj);
        PyObject *descr = NULL;
        PyObject *res = NULL;
        
        ...
        descr = _PyType_Lookup(tp, name);

        Py_XINCREF(descr);

        f = NULL;
        if (descr != NULL &&
            PyType_HasFeature(descr->ob_type, Py_TPFLAGS_HAVE_CLASS)) {
            f = descr->ob_type->tp_descr_get;
            if (f != NULL && PyDescr_IsData(descr)) {
                res = f(descr, obj, (PyObject *)obj->ob_type);
                Py_DECREF(descr);
                goto done;
            }
        }

        ...
        return res;
    }

    PyObject *
    _PyType_Lookup(PyTypeObject *type, PyObject *name)
    {
        Py_ssize_t i, n;
        PyObject *mro, *res, *base, *dict;
        unsigned int h;

        ...

        /* Look in tp_dict of types in MRO */
        mro = type->tp_mro;

        ...

        res = NULL;
        assert(PyTuple_Check(mro));
        n = PyTuple_GET_SIZE(mro);
        for (i = 0; i < n; i++) {
            base = PyTuple_GET_ITEM(mro, i);
            if (PyClass_Check(base))
                dict = ((PyClassObject *)base)->cl_dict;
            else {
                assert(PyType_Check(base));
                dict = ((PyTypeObject *)base)->tp_dict;
            }
            assert(dict && PyDict_Check(dict));
            res = PyDict_GetItem(dict, name);
            if (res != NULL)
                break;
        }

        ...
        return res;
    }


`_PyObject_GenericGetAttrWithDict`中先调用`_PyType_Lookup`从`Foo`的`tp_mro`中查到`__class__`属性(来自`Foo`的基类`object`)，该属性是一个`data descriptor`，最终调用了`object_get_class`。  

**typeobject.c**

    static PyObject *
    object_get_class(PyObject *self, void *closure)
    {
        Py_INCREF(Py_TYPE(self));
        return (PyObject *)(Py_TYPE(self));
    }

从`object_get_class`函数可以看出，对于`f = Foo(); f.__class__;`来说也是返回的`Foo`对象的`ob_type`。这就解释了**Case 1**中为什么`type(b)`和`b.__class__`是相等的。
___

### 那为什么**Case 2**中的`type(a)`和`a.__class__`不相等呢？
因为**Case 1**中没有自定义`__class__`，所以查找`__class__`时在`Bar`中没找到，接着就去`Bar`的基类`object`中找，正好`object`中定义了一个`__class__`的 data descriptor, 就返回这个。  

而在**Case 2**中我们自定义了`__class__`，所以在`ASML.__dict__`中找到有这个 attribute 后就返回了，不会再去找`mro`中的下一个(`object`)。但是这里找到的这个 attribute 不是 data descriptor，根据前面提到的 attribute 搜索顺序，我们接着在`a.__dict__`中找，也没有，那就直接返回`ASML`中的找到的那个了。
___

### 为什么设置__class__后，Case 1和Case 2有差别
**obj.xxx = yyy 设置attribute时的顺序**  
 1. 先从type对象及其基类的`__dict__`中查找该 attribute，找到就返回。如果找到的是 data descriptor，则用该 data descriptor的`__set__`来设置  
 2. 否则添加到`obj.__dict__`里   

**Case 1**中得到的`__class__`是一个 data descriptor，给它赋值实际上调用的是`object_set_class`函数。

**typeobject.c**

    static int
    object_set_class(PyObject *self, PyObject *value, void *closure)
    {
        PyTypeObject *oldto = Py_TYPE(self);
        PyTypeObject *newto;

        ...
        newto = (PyTypeObject *)value;
        ...
        if (compatible_for_assignment(newto, oldto, "__class__")) {
            Py_INCREF(newto);
            Py_TYPE(self) = newto;
            Py_DECREF(oldto);
            return 0;
        }
        else {
            return -1;
        }
    }

从该函数的实现可以看到`b.__class__ = Foo`实际上会把`b`的`ob_type`设为`Foo`。所以赋值后`type(b)`和`b.__class__`都跟着变了。

    >>> b.__dict__
    {}
    >>> Bar.__dict__
    dict_proxy({'__dict__': <attribute '__dict__' of 'Bar' objects>, '__module__': '__main__', '__weakref__': <attribute '__weakref__' of 'Bar' objects>, '__doc__': None})
    >>>
    >>> b.__class__ = Foo
    >>>
    >>> b.__class__, type(b)
    (<class '__main__.Foo'>, <class '__main__.Foo'>)
    >>> b.__dict__
    {}
    >>> Bar.__dict__
    dict_proxy({'__dict__': <attribute '__dict__' of 'Bar' objects>, '__module__': '__main__', '__weakref__': <attribute '__weakref__' of 'Bar' objects>, '__doc__': None})


而**Case 2**中得到的`__class__`不是 data descriptor。所以`a.__class__ = Foo`会在`a.__dict__`中添加一条记录，而`a`的`ob_type`不会变，`ASML.__dict__`也不会变。

    >>> a.__dict__
    {}
    >>> ASML.__dict__
    dict_proxy({'__dict__': <attribute '__dict__' of 'ASML' objects>, '__module__': '__main__', '__weakref__': <attribute '__weakref__' of 'ASML' objects>, '__class__': <class '__main__.Foo'>, '__doc__': None})
    >>>
    >>> a.__class__ = Brion
    >>>
    >>> a.__dict__
    {'__class__': <class '__main__.Brion'>}
    >>> ASML.__dict__
    dict_proxy({'__dict__': <attribute '__dict__' of 'ASML' objects>, '__module__': '__main__', '__weakref__': <attribute '__weakref__' of 'ASML' objects>, '__class__': <class '__main__.Foo'>, '__doc__': None})

___
以上的讨论都是基于**new style class**。对于**old style class**来说, `type()`不等于`__class__`:

    >>> class A():
        pass

    >>> a = A()
    >>> a.__class__, type(a)
    (<class __main__.A at 0x0270CFB8>, <type 'instance'>)


另外一个关于 isintance(obj, cls) 的问题
---

    >>> class Foo(object):
        pass
        
    >>> class ASML(object):
        __class__ = Foo
        
    >>> a = ASML()
    >>> isinstance(a, Foo)
    True
    >>> isinstance(a, ASML)
    True

为什么这里的两个`isinstance`都返回`True`呢？  

**object.h**

    #define Py_TYPE(ob)             (((PyObject*)(ob))->ob_type)

    #bltinmodule.c
    static PyMethodDef builtin_methods[] = {
        ...
        {"isinstance",  builtin_isinstance, METH_VARARGS, isinstance_doc},
        ...
    }

    static PyObject *
    builtin_isinstance(PyObject *self, PyObject *args)
    {
        PyObject *inst;
        PyObject *cls;
        int retval;

        if (!PyArg_UnpackTuple(args, "isinstance", 2, 2, &inst, &cls))
            return NULL;

        retval = PyObject_IsInstance(inst, cls);
        if (retval < 0)
            return NULL;
        return PyBool_FromLong(retval);
    }

最终`isinstance(obj, cls)`会调用`PyObject_IsInstance`。

**abstract.c**

    int
    PyObject_IsInstance(PyObject *inst, PyObject *cls) // Yun: "isinstance(obj, cls)" will call it
    {
        static PyObject *name = NULL;

        /* Quick test for an exact match */
        if (Py_TYPE(inst) == (PyTypeObject *)cls)      // Yun: "isinstacne(b, ASML)" returns True
            return 1;

        ...
        if (!(PyClass_Check(cls) || PyInstance_Check(cls))) {
            PyObject *checker;
            checker = _PyObject_LookupSpecial(cls, "__instancecheck__", &name);
            ...
            res = PyObject_CallFunctionObjArgs(checker, inst, NULL);  // Yun: "isinstance(b, Foo)" call recursive_isinstance() and returns True
            if (res != NULL) {
                ok = PyObject_IsTrue(res);
                ...
            }
            return ok;
        }
        return recursive_isinstance(inst, cls);        
    }

    static int
    recursive_isinstance(PyObject *inst, PyObject *cls)
    {
        ...
        static PyObject *__class__ = NULL;
        int retval = 0;
        if (__class__ == NULL) {
            __class__ = PyString_InternFromString("__class__");
            if (__class__ == NULL)
                return -1;
        }
        ...

        if (PyClass_Check(cls) && PyInstance_Check(inst)) {
            ...
        }
        else if (PyType_Check(cls)) {
            retval = PyObject_TypeCheck(inst, (PyTypeObject *)cls);
            if (retval == 0) {
                PyObject *c = PyObject_GetAttr(inst, __class__);
                ...
                retval = PyType_IsSubtype(
                        (PyTypeObject *)c,
                        (PyTypeObject *)cls);      // Yun: Both "c" and "cls" are "Foo" here
        }
        ...

        return retval;
    }


从`__instancecheck__` 到 `recursive_isinstance`:

**typeobject.c**

    static PyMethodDef type_methods[] = {
        {"mro", (PyCFunction)mro_external, METH_NOARGS,
         PyDoc_STR("mro() -> list\nreturn a type's method resolution order")},
        {"__subclasses__", (PyCFunction)type_subclasses, METH_NOARGS,
         PyDoc_STR("__subclasses__() -> list of immediate subclasses")},
        {"__instancecheck__", type___instancecheck__, METH_O,
         PyDoc_STR("__instancecheck__() -> bool\ncheck if an object is an instance")},
        {"__subclasscheck__", type___subclasscheck__, METH_O,
         PyDoc_STR("__subclasscheck__() -> bool\ncheck if a class is a subclass")},
        {0}
    };

    static PyObject *
    type___instancecheck__(PyObject *type, PyObject *inst)
    {
        switch (_PyObject_RealIsInstance(inst, type)) {
        case -1:
            return NULL;
        case 0:
            Py_RETURN_FALSE;
        default:
            Py_RETURN_TRUE;
        }
    }

    int
    _PyObject_RealIsInstance(PyObject *inst, PyObject *cls)
    {
        return recursive_isinstance(inst, cls);
    }

简单来说就是`isinstance(obj, cls)`会先看`ob_type`，然后在看`__class__`。