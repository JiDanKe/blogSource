title: "python属性查找"
date: 2015-04-21 16:59:34
tags: [python, descriptor, MRO, __dict__, __slots__]
categories: python
---
#### 1 摘要

介绍python属性查找的相关基础知识，通过底层代码的分析，详细描述了python中对象属性的查找/赋值过程以及在object和type中对描述器的不同处理。并结合这些知识，简单探寻了一下描述器在python实现中的作用。

#### 2 准备

在详解python属性查找之前，我们先来了解一些相关的概念，以便于理解整个查找过程。

##### 2.1 描述器

描述器(descriptor )是一个有 “绑定行为” 的对象属性(object attribute)，它的访问控制被描述器协议方法重写。描述器协议的方法为：

    descr.__get__(self, obj, type=None) --> value
    descr.__set__(self, obj, value) --> None
    descr.__delete__(self, obj) --> None

一个对象具有其中任一个方法就会成为描述器，从而在被当作**对象属性**时重写默认的查找行为。

如果一个对象同时定义了 ** \_\_get\_\_() ** 和 ** \_\_set\_\_()** ,它叫做资料描述器(data descriptor)。仅定义了 ** \_\_get\_\_() ** 的描述器叫非资料描述器(non-data descriptor, 常用于方法，当然其他用途也是可以的)。

资料描述器和非资料描述器的区别在于：相对于实例的字典的优先级。如果实例字典中有与描述器同名的属性，如果描述器是资料描述器，优先使用资料描述器，如果是非资料描述器，优先使用字典中的属性。后面内容会具体介绍。

描述器是强大的，应用广泛的。描述器正是属性, 实例方法, 静态方法, 类方法和 super 的背后的实现机制。描述器在Python自身中广泛使用，以实现Python 2.2中引入的新式类。描述器简化了底层的C代码，并为Python的日常编程提供了一套灵活的新工具。

##### 2.2 Python方法解析顺序

在支持多重继承的编程语言中，查找方法具体来自那个类时的基类搜索顺序通常被称为方法解析顺序(Method Resolution Order)，简称MRO。Python中查找属性也遵循同一规则。对于仅支持单重继承的语言，MRO非常简单。但是在支持多继承的语言中，情况就比较复杂了。Python的历史版本中先后出现三种不同的MRO：经典方式、Python2.2 新式算法、Python2.3 新式算法(也称作C3)。Python 3中只保留了最后一种，即C3算法。

python2.2为解决引入新类所带来的方法解析顺序，采用的方案是在类定义时就计算出它的MRO，并存储为该类对象的一个属性 ** \_\_mro\_\_ ** 。然后在查找属性时按照 ** \_\_mro\_\_ ** 依次搜索基类。

##### 2.3 metaclass

在大多数编程语言中，类就是一组用来描述如何生成一个对象的代码段。Python中的类还远不止如此。在python中类同样也是一种对象，只要你使用关键字class，解释器在执行的时候就会创建一个对象。例如：

```python
class ObjectCreator(object):pass

```

上面的代码会在python解释器执行的时候生成一个对象 **ObjectCreator**。***这个对象（类）自身拥有创建对象（类实例）的能力，而这就是为什么它是一个类的原因。*** 创建 “类” 的 “类” 就是元类（metaclass），python默认的内建元类是type。

```python
>>> object.__class__
<type 'type'>
>>> type(object)
<type 'type'>
```

在python中通过type(obj)，或者 obj.\_\_class\_\_ 来获取对象的类型（type）。这里有个特别的type的type是type。

```python
>>> type(object)
<type 'type'>
>>> type(type)
<type 'type'>
>>> type.__class__
<type 'type'>
>>> object.__class__
<type 'type'>
```

换句话说，元类（metaclass）的实例化是类（class），类（class）的实例化是类实例对象（object）。

<!-- more -->

#### 3 属性的查找过程

为了描述方便，我们假定现在要访问 b.x。x不是python内建的特殊属性。**（注：如果查找的属性是一个python内建的特殊属性，直接就找到，不需要执行下面的过程！）**

在python中属性的查找由类型的内部方法 **\_\_getattribute\_\_()** 支持。由于当b为object或者class时，**\_\_getattribute\_\_()** 对于描述器的调用方式不同，为了更好地介绍其细节，我们将查找过程按照b为object与class(为了与内建元类type区别，这里不用type而用class代替)时进行分开描述（cpython中实际对应的底层c代码实现也不同，具体细节后面会介绍）。


##### 3.1 object属性的查找过程：object.\_\_getattribute\_\_()

1. 沿着 **type(b).\_\_mro\_\_** 搜索基类 **\_\_dict\_\_** 中名称为 x 的属性，并将其值赋值给 **descr** 变量（descr默认为null）；

2. 若 descr 是一个 data descriptor 则执行 `descr.__get__(b, type(b))`,并将执行结果返回，结束查找，否则进入下一步；

3. 在 **b.\_\_dict\_\_** 中查找名称为 x 的属性，若找到则将其返回，结束查找，否则进入下一步；

4. 若上述第2步查找失败（descr == null），则抛出 AttributeError 异常，结束查找。若descr（descr != null）是 non-data descriptor 则执行 `descr.__get__(b, type(b))`,并将执行结果返回，结束查找；否则直接返回 descr， 结束查找。


cpython 对应的底层实现代码： `PyObject_GenericGetAttr()` in [Objects/object.c](https://hg.python.org/cpython/file/2.7/Objects/object.c)。

```c
PyObject *
PyObject_GenericGetAttr(PyObject *obj, PyObject *name)
{
    return _PyObject_GenericGetAttrWithDict(obj, name, NULL);
}

PyObject *
_PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name, PyObject *dict)
{
    PyTypeObject *tp = Py_TYPE(obj);/*对象的类型*/
    PyObject *descr = NULL;
    PyObject *res = NULL;
    descrgetfunc f;
    Py_ssize_t dictoffset;
    PyObject **dictptr;

    if (!PyString_Check(name)){
#ifdef Py_USING_UNICODE
        /* The Unicode to string conversion is done here because the
           existing tp_setattro slots expect a string object as name
           and we wouldn't want to break those. */
        if (PyUnicode_Check(name)) {
            name = PyUnicode_AsEncodedString(name, NULL, NULL);
            if (name == NULL)
                return NULL;
        }
        else
#endif
        {
            PyErr_Format(PyExc_TypeError,
                         "attribute name must be string, not '%.200s'",
                         Py_TYPE(name)->tp_name);
            return NULL;
        }
    }
    else
        Py_INCREF(name);

    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            goto done;
    }

#if 0 /* XXX this is not quite _PyType_Lookup anymore */
    /* Inline _PyType_Lookup*/
    {
        Py_ssize_t i, n;
        PyObject *mro, *base, *dict;

        /* Look in tp_dict of types in MRO */
        mro = tp->tp_mro;
        assert(mro != NULL);
        assert(PyTuple_Check(mro));
        n = PyTuple_GET_SIZE(mro);
        for (i = 0; i < n; i++) {
            base = PyTuple_GET_ITEM(mro, i);
            if (PyClass_Check(base))/*检查是否是metaclss*/
                dict = ((PyClassObject *)base)->cl_dict;
            else {
                assert(PyType_Check(base));
                dict = ((PyTypeObject *)base)->tp_dict;
            }
            assert(dict && PyDict_Check(dict));
            descr = PyDict_GetItem(dict, name);
            if (descr != NULL)
                break;
        }
    }
    /*内联_PyType_Lookup方法，可以看到其是在tp_dict中搜索*/
#else
    descr = _PyType_Lookup(tp, name); /*沿着mro搜索基类的__dict__*/
#endif

    Py_XINCREF(descr);

    f = NULL;

    /*搜索到属性且为data descriptor，执行__get__(),返回执行结果，结束查找*/
    if (descr != NULL &&
        PyType_HasFeature(descr->ob_type, Py_TPFLAGS_HAVE_CLASS)) {
        f = descr->ob_type->tp_descr_get;
        if (f != NULL && PyDescr_IsData(descr)) {
            res = f(descr, obj, (PyObject *)obj->ob_type);
            Py_DECREF(descr);
            goto done;
        }
    }

    /*移动指针到实例的__dict__，准备搜索。
     *如果类定义了__slots__属性，则tp_dictoffset=0，
     *详见Objects/typeobject.c中type_new方法。
     *由于没有为实例分配__dict__空间，所以不能为实例新增属性，也节省了创建实例的内存开销。
     *这就是__slots__的原理和效果。
     */
    if (dict == NULL) {
        /* Inline _PyObject_GetDictPtr */
        dictoffset = tp->tp_dictoffset;
        if (dictoffset != 0) {
            if (dictoffset < 0) {
                Py_ssize_t tsize;
                size_t size;

                tsize = ((PyVarObject *)obj)->ob_size;
                if (tsize < 0)
                    tsize = -tsize;
                size = _PyObject_VAR_SIZE(tp, tsize);

                dictoffset += (long)size;
                assert(dictoffset > 0);
                assert(dictoffset % SIZEOF_VOID_P == 0);
            }
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        }
    }

    /*若在实例__dict__中搜索到属性，则返回属性，结束查找。
     * 一定要住注意这里与type.__getattribute__()的区别，在object.__dict__中找到的
     * 属性是直接返回的，直接返回。所以如果为一个实例增加一个descriptor属性，是不会
     * 被object.__getattribute__()执行的。
     */
    if (dict != NULL) {
        Py_INCREF(dict);
        res = PyDict_GetItem(dict, name);
        if (res != NULL) {
            Py_INCREF(res);
            Py_XDECREF(descr);
            Py_DECREF(dict);
            goto done;
        }
        Py_DECREF(dict);
    }

    /*沿mro搜索到的属性值若是定义了 __get__() 方法的 descriptor，
     *则执行__get__()，返回结果，结束查找
     */
    if (f != NULL) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        Py_DECREF(descr);
        goto done;
    }

    /*沿mro搜索到的属性值不是descriptor，返回结果，结束查找*/
    if (descr != NULL) {
        res = descr;
        /* descr was already increfed above */
        goto done;
    }

    /*沿mro也没有搜索则抛出异常，结束查找*/
    PyErr_Format(PyExc_AttributeError,
                 "'%.50s' object has no attribute '%.400s'",
                 tp->tp_name, PyString_AS_STRING(name));
  done:
    Py_DECREF(name);
    return res;
}

```

##### 3.2 class属性的查找过程：type.\_\_getattribute\_\_()

1. 沿着 **type(b).\_\_mro\_\_** 搜索(元类)基类 **\_\_dict\_\_** 中名称为 x 的属性，并将其值赋值给 **meta_atrribute** 变量（meta_atrribute默认为null）；

2. 若 meta_atrribute 是一个 data descriptor 则执行 `meta_atrribute.__get__(b, type(b))`,并将执行结果返回，结束查找，否则进入下一步；

3.  在 **b.\_\_dict\_\_** 中查找名称为 x 的属性，若没有找到，则进入下一步。若找到，当属性为 descriptor 时执行 `meta_atrribute.__get__(None, b)`， 并将其结果返回，否则直接将属性返回，结束查找。

4. 若上述第2步查找失败（meta_atrribute == null），则抛出 AttributeError 异常，结束查找。meta_atrribute（meta_atrribute != null）是 non-data descriptor 则执行 `meta_atrribute.__get__(b, type(b))`,并将执行结果返回，结束查找；否则直接返回 descr， 结束查找。

cpython对应的底层实现代码： `type_getattro()` in [Objects/typeobject.c](https://hg.python.org/cpython/file/2.7/Objects/typeobject.c)。
```c
/* This is similar to PyObject_GenericGetAttr(),
   but uses _PyType_Lookup() instead of just looking in type->tp_dict. */
static PyObject *
type_getattro(PyTypeObject *type, PyObject *name)
{
    PyTypeObject *metatype = Py_TYPE(type);
    PyObject *meta_attribute, *attribute;
    descrgetfunc meta_get;

    if (!PyString_Check(name)) {
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     name->ob_type->tp_name);
        return NULL;
    }

    /* Initialize this type (we'll assume the metatype is initialized) */
    if (type->tp_dict == NULL) {
        if (PyType_Ready(type) < 0)
            return NULL;
    }

    /* No readable descriptor found yet */
    meta_get = NULL;

    /* Look for the attribute in the metatype */
    meta_attribute = _PyType_Lookup(metatype, name);

    /*如果在元类中找到对应名称的data descriptor属性，则执行 __get__()方法，
     *将结果返回，结束查找
     */
    if (meta_attribute != NULL) {
        meta_get = Py_TYPE(meta_attribute)->tp_descr_get;

        if (meta_get != NULL && PyDescr_IsData(meta_attribute)) {
            /* Data descriptors implement tp_descr_set to intercept
             * writes. Assume the attribute is not overridden in
             * type's tp_dict (and bases): call the descriptor now.
             */
            return meta_get(meta_attribute, (PyObject *)type,
                            (PyObject *)metatype);
        }
        Py_INCREF(meta_attribute);
    }

    /* No data descriptor found on metatype. Look in tp_dict of this
     * type and its bases */
    attribute = _PyType_Lookup(type, name);
    if (attribute != NULL) {
        /* Implement descriptor functionality, if any */
        descrgetfunc local_get = Py_TYPE(attribute)->tp_descr_get;

        Py_XDECREF(meta_attribute);

        /*在类的__dict__中找到定义了  __get__() 方法的descriptor属性，则执行
         * __get__()方法，将结果返回，结束查找。
         *
         * 这里与在object.__dict__中查找到属性后的处理方式不同：在object中直接
         * 将属性属性返回；在type.__dict__中找到属性后如果属性是定义了 __get__()
         * 方法的descriptor，要执行__get__(None, type)返回结果。
         */
        if (local_get != NULL) {
            /* NULL 2nd argument indicates the descriptor was
             * found on the target object itself (or a base)  */
            return local_get(attribute, (PyObject *)NULL,
                             (PyObject *)type);
        }

        /*在类__dict__中找到的属性不是descriptor时，直接将属性返回，结束查找*/
        Py_INCREF(attribute);
        return attribute;
    }

     /*若前面没有在元类中找到属性，直接抛出异常，结束查找；
      *若在元类中找到的属性不是descriptor，直接返回，结束查找。
      *若在元类中找到的属性是non-data descriptor, 则执行 __get__()方法，
      *将结果返回，结束查找。
      */

    /* No attribute found in local __dict__ (or bases): use the
     * descriptor from the metatype, if any */
    if (meta_get != NULL) {
        PyObject *res;
        res = meta_get(meta_attribute, (PyObject *)type,
                       (PyObject *)metatype);
        Py_DECREF(meta_attribute);
        return res;
    }

    /* If an ordinary attribute was found on the metatype, return it now */
    if (meta_attribute != NULL) {
        return meta_attribute;
    }

    /* Give up */
    PyErr_Format(PyExc_AttributeError,
                     "type object '%.50s' has no attribute '%.400s'",
                     type->tp_name, PyString_AS_STRING(name));
    return NULL;
}

```

##### 3.3 在 `object.__getattribute__()` 与 `type.__getattribute__()` 对descriptor不同处理

由上面的介绍可见，`object.__getattribute__()` 与 `type.__getattribute__() `在实例字典（`object.__dict__, type.__dict__`) 找到的descriptor类型属性的处理方式不同（步骤3）。object中是直接返回，type中会用调用\_\_get\_\_(None, type),返回结果（注意调用的参数）。

之前在阅读python描述器的时候，读到object与type的\_\_getattribute\_\_()方法对描述器的处理方式区别，作者用pure python代码描述了用object.\_\_getattribute\_\_()等价实现的type.\_\_getaatribute\_\_()一直不能理解。现在结合前述解析，应该很好理解了，注意代码中self实际是一个cls（type）：

```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
       return v.__get__(None, self)
    return v
```

##### 3.4 属性赋值时的查找过程

python中对于object和type的属性赋值查找过程是相同的。Objects/typeobject.c 的type_setattro()方法最终调用也是 Objects/object.c 的 **PyObject_GenericSetAttr()**方法。如下c的 **type_setattro()** 代码所示：

```c
static int
type_setattro(PyTypeObject *type, PyObject *name, PyObject *value)
{
    if (!(type->tp_flags & Py_TPFLAGS_HEAPTYPE)) {
        PyErr_Format(
            PyExc_TypeError,
            "can't set attributes of built-in/extension type '%s'",
            type->tp_name);
        return -1;
    }
    if (PyObject_GenericSetAttr((PyObject *)type, name, value) < 0)
        return -1;
    return update_slot(type, name);
}
```

Objects/object.c 的 **PyObject_GenericSetAttr()**中对属性赋值的过程是：

1. 在type(obj) [**注：对于object是class，对于class是metaclass**] 的继承链中查找属性，若找到的属性是data descriptor，则调用\_\_set\_\_(obj, value)方法设置属性，操作完成返回。否则进去下一步。

2. 如果obj.\_\_dict\_\_存在，直接在obj.\_\_dict\_\_中加入属性，操作完成返回。否则进去下一步。

3. 若在第1步type(obj)的继承链中找到的属性是descriptor，且定义了\_\_set\_\_方法，则调用\_\_set\_\_(obj, value)方法设置属性，操作完成返回。

4. 抛出异常属性只读异常（比如用@property装饰的属性）。

```c
int
PyObject_GenericSetAttr(PyObject *obj, PyObject *name, PyObject *value)
{
    return _PyObject_GenericSetAttrWithDict(obj, name, value, NULL);
}

int
_PyObject_GenericSetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *value, PyObject *dict)
{
    PyTypeObject *tp = Py_TYPE(obj);
    PyObject *descr;
    descrsetfunc f;
    PyObject **dictptr;
    int res = -1;

    if (!PyString_Check(name)){
#ifdef Py_USING_UNICODE
        /* The Unicode to string conversion is done here because the
           existing tp_setattro slots expect a string object as name
           and we wouldn't want to break those. */
        if (PyUnicode_Check(name)) {
            name = PyUnicode_AsEncodedString(name, NULL, NULL);
            if (name == NULL)
                return -1;
        }
        else
#endif
        {
            PyErr_Format(PyExc_TypeError,
                         "attribute name must be string, not '%.200s'",
                         Py_TYPE(name)->tp_name);
            return -1;
        }
    }
    else
        Py_INCREF(name);

    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            goto done;
    }

    /*在type(obj)的继承链中查找属性*/
    descr = _PyType_Lookup(tp, name);
    f = NULL;
    /*若找到的属性是data descriptor则执行__set__操作后结束赋值操作，返回。*/
    if (descr != NULL &&
        PyType_HasFeature(descr->ob_type, Py_TPFLAGS_HAVE_CLASS)) {
        f = descr->ob_type->tp_descr_set;
        if (f != NULL && PyDescr_IsData(descr)) {
            res = f(descr, obj, value);
            goto done;
        }
    }

    if (dict == NULL) {
        dictptr = _PyObject_GetDictPtr(obj);
        if (dictptr != NULL) {
            dict = *dictptr;
            if (dict == NULL && value != NULL) {
                dict = PyDict_New();
                if (dict == NULL)
                    goto done;
                *dictptr = dict;
            }
        }
    }
    /*若在obj中有__dict__则直接将属性赋值*/
    if (dict != NULL) {
        Py_INCREF(dict);
        if (value == NULL)
            res = PyDict_DelItem(dict, name);
        else
            res = PyDict_SetItem(dict, name, value);
        if (res < 0 && PyErr_ExceptionMatches(PyExc_KeyError))
            PyErr_SetObject(PyExc_AttributeError, name);
        Py_DECREF(dict);
        goto done;
    }

    /*在type(obj)的继承链中查找到的属性是可写的（定义了__set__ 方法），执行__set__,结束属性赋值，返回。*/
    if (f != NULL) {
        res = f(descr, obj, value);
        goto done;
    }

    /*抛出属性只读异常*/
    if (descr == NULL) {
        PyErr_Format(PyExc_AttributeError,
                     "'%.100s' object has no attribute '%.200s'",
                     tp->tp_name, PyString_AS_STRING(name));
        goto done;
    }

    PyErr_Format(PyExc_AttributeError,
                 "'%.50s' object attribute '%.400s' is read-only",
                 tp->tp_name, PyString_AS_STRING(name));
  done:
    Py_DECREF(name);
    return res;
}
```


##### 3.5 `object.__getattr__(self, name)` 与 `object.__getattribute__(self, name)`

通过自定义object.\_\_getattr\_\_和object.\_\_getattribute\_\_方法，我们可以控制对象属性访问过程。一般而言，访问对象属性时首先调用object.\_\_getattribute\_\_方法，当该方法抛出 AttributeError 异常时，调用object.\_\_getattr\_\_方法，并返回调用结果。

1. **object.\_\_getattribute\_\_(self, name)**
    在访问类实例的属性时无条件调用这个方法。 如果类也定义了方法 \_\_getattr\_\_()，那么除非 \_\_getattribute\_\_() 显式地调用了它，或者抛出了 AttributeError 异常， 否则它就不会被调用。 这个方法应该返回一个计算好的属性值， 或者抛出异常 AttributeError。为了避免无穷递归，对于任何它需要访问的属性， 这个方法应该调用基类的同名方法，例如, object.\_\_getattribute\_\_(self, name).

    注意：通过特定语法或者内建函式， 做隐式调用搜索特殊方法时，这个方法可能会被跳过，参见 搜索特殊方法。

2. **object.\_\_getattr\_\_(self, name)**
    在正常方式访问属性无法成功时 (就是说, self属性既不是实例的, 在类树结构中找不到) 使用。 name 是属性名，该方法应该返回一个计算好的属性值或抛出一个 AttributeError 异常。

    注意， 如果属性可以通过正常方法访问， \_\_getattr\_\_() 是不会被调用的 (是有意将 \_\_getattr\_\_() 和 \_\_setattr\_\_() 设计成不对称的)。这样做的原因是基于效率的考虑，并且这样也不会让 \_\_getattr\_\_() 干涉正常属性。 注意， 至少对于类实例而言， 不必非要更新实例字典伪装属性 (但可以将它们插入到其它对象中)。需要全面控制属性访问，可以参考以下 \_\_getattribute\_\_() 的介绍.


对于对象属性赋值的控制，python提供了object.\_\_setattr\_\_方法：
1. **object.\_\_setattr\_\_(self, name, value)**

    在属性要被赋值时调用。这会替代正常机制 (即把值保存在实例字典中)。name 是属性名， vaule 是要赋的值.

    如果在 \_\_setattr\_\_() 里要对一个实例属性赋值， 它应该调用父类的同名方法，例如， object.\_\_setattr\_\_(self, name, value)。

##### 3.6 \_\_dict\_\_ 与 \_\_slots\_\_

python在实例化一个类时，会为实例创建一个字典用来存储实例属性。该字典通过属性 `__dict__` 暴露出来的， 其背后实现的技术是descriptor——在类的 \_\_dict\_\_  中定义了一个名叫 ”\_\_dict\_\_“ 的data descriptor。结合之前的属性访问机制，我们很容易理解下面的代码：

```python
>>> class A(object):pass
...


>>> A.__dict__
dict_proxy({'__dict__': <attribute '__dict__' of 'A' objects>, '__module__': '__
main__', '__weakref__': <attribute '__weakref__' of 'A' objects>, '__doc__': Non
e})
>>> type(A.__dict__["__dict__"]) # 类A的__dict__属性中有一个 ”__dict__“ data descriptor
<type 'getset_descriptor'>

# 通过使用类A的实例a调用该描述器的__get__方法得到与直接访问A().__dict__相同的结果（参考3.3的内容）
>>> a=A()
>>> A.__dict__["__dict__"].__get__(a,A)
{}
>>> a.__dict__
{}
>>> A.__dict__["__dict__"].__get__(a,A) == a.__dict__
True
>>>
```

***注：也就是说，python提供了另外一种方式来操作实例的属性：通过 `obj.__dict__` 访问到实例字典，通过操作该字典操作实例的属性。***

python默认每创建一个实例都会为该实例分配一个用于保存实例属性的字典。**对于那些没有什么实例属性且会创建大量实例的类型而言，这种默认方式会造成不必要的内存浪费**。考虑到这个问题，python在新式类中引入了\_\_slots\_\_属性。当一个类定义了 \_\_slots\_\_ 属性，在实例化该类时便不会为实例分配用于保存实例属性的字典，而是预先分配为 \_\_slots\_\_ 声明的属性分配好空间。其背后实现的技术也是descriptor。

```python
>>> class A(object):__slots__ = ("s",)
...
>>> A.__dict__
dict_proxy({'s': <member 's' of 'A' objects>, '__module__': '__main__', '__slots
__': ('s',), '__doc__': None}) # 类A定义了__slots__属性，其 __dict__ 中没有包含支持实例属性的 "__dict__" 资料描述器


>>> a = A()
>>> a.__dict__ # 类A定义了__slots__属性，没有为实例分配实例字典
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'A' object has no attribute '__dict__'

>>> type(A.__dict__["s"]) # __slots__ 声明的属性，其背后实现的技术也是descriptor
<type 'member_descriptor'>
>>>

```

有一个需要注意的问题，在定义了 \_\_slots\_\_的类型中，若没有在其实例上定义相应的属性（为属性赋值），那么其背后的 `member_descriptor` 调用会抛出  AttributeError 异常。换句话说，定义了 \_\_slots\_\_ 的类型实例化时并不会为属性提供默认值。

```python
>>> a.s # 没有在实例上为属性赋值，访问会出错
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: s
>>> a.s = 10
>>> a.s
10
>>>
```

#### 4 结束语

通过对python属性查找过程的解析，我们初步了解了其背后实现的机制，对于深入理解这门语言是大有裨益的。关于metaclass、MRO、Descriptor更深入的内容，可参考后面的资料。

我使用python也有一段时间了，但是对语言背后的实现细节一直没有系统地研究过。每一次涉及到相关的问题，都是浅尝辄止，总是抱着问题解决就结束的态度。也就导致一次一次重复去查阅相关资料，浪费了时间，也没有真正理解。对其中很多底层技术和概念都有所了解，但没有深入，也没有思考过其中的联系。借由这次遇到的问题，整理了一下，这篇文字算是一个总结。

#### 5 参考资料

1. Guido van Rossum 撰写的 [Method Resolution Order](http://python-history.blogspot.com/2010/06/method-resolution-order.html) 详细介绍了python的MRO算法演变历史。

2. Michele Simionato 撰写的 [The Python 2.3 Method Resolution Order](https://www.python.org/download/releases/2.3/mro/) 详细介绍了python2.3 MRO采用的C3算法。

3. e-satis 在Stack Overflow回答提问 [What is a metaclass in Python?](http://stackoverflow.com/questions/100003/what-is-a-metaclass-in-python) 的经典回答，通俗易懂地讲解了python的metaclass。

4. python文档中详细介绍了 [\_\_slots\_\_](http://docs.python.org/2/reference/datamodel.html?highlight=__slots__#__slots__)。
