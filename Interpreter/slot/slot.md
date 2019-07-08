# slot

# contents

* [related file](#related-file)
* [slot](#slot)
* [example](#example)
* [instance access](#instance-access)
	* [instance access wing](#instance-access-wing)
		* [before set a value](#before-set-a-value)
		* [after set a value](#after-set-a-value)
	* [instance access x](#instance-access-x)
* [type-access](#type-access)
	* [type access wing](#type-access-wing)
	* [type access x](#type-access-x)
* [difference](#difference)
	* [with slot](#with-slot)
		* [how does attributes initialized in the creation of `class A`](#how-does-attributes-initialized-in-the-creation-of-class-A-?)
		* [how does attributes initialized in the creation of `instance a` ?](#how-does-attributes-initialized-in-the-creation-of-instance-a-?)
		* [lookup procedure in MRO ?](#lookup-procedure-in-MRO-?)
	* [without slot](#without-slot)
		* [how does attributes initialized in the creation of `class A`](#how-does-attributes-initialized-in-the-creation-of-class-A-?)
		* [how does attributes initialized in the creation of `instance a` ?](#how-does-attributes-initialized-in-the-creation-of-instance-a-?)
		* [lookup procedure in MRO ?](#lookup-procedure-in-MRO-?)
	* [memory saving measurement](#memory-saving-measurement)
* [read more](#read-more)

# related file
* cpython/Objects/typeobject.c
* cpython/Objects/clinic/typeobject.c.h
* cpython/Include/cpython/object.h


# slot

# example

    class A(object):
        __slots__ = ["wing", "leg"]

        x = 3


what's the difference of accessing attribute `wing` and `x` of instance `a` ?

what's the difference of accessing attribute `wing` and `x` of type `A` ?

# instance access

## instance access wing

### before set a value

	>>> a = A()
    >>> a.wing
    Traceback (most recent call last):
      File "<input>", line 1, in <module>
    AttributeError: wing

according to the **attribute accessing procedure** described in [descr](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/descr/descr.md)

we can draw the procedure of accessing `a.wing` in  a brief level

![instance_desc](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/instance_desc.png)

the **descriptor protocol** help us find an object named `descr`, type is `member_descriptor`, if you call

	repr(descr)
    descr: <member 'wing' of 'A' objects>

the `descr` has a member named `d_member`, which stores information about the attribute name, type and offset

with the informations stored in `d_member`, you are able to access the address of attribute in a very fast way

in the current example, if the begin address of the instance `a` is `0x00`, address after adding the offset is `0x18`, you can cast the address in `0x18` to a proper object according to `type` stored in `d_member`

	/* Include/structmember.h */
    #define T_OBJECT_EX 16  /* Like T_OBJECT, but raises AttributeError
                               when the value is NULL, instead of
                               converting to None. */

	/* Python/structmember.c */
    addr += l->offset;
    switch (l->type) {
    	/* ... */
        case T_OBJECT_EX:
            v = *(PyObject **)addr;
            /* because wing attribute of instance a is never set,
            it raise an attribute error */
            if (v == NULL)
                PyErr_SetString(PyExc_AttributeError, l->name);
            Py_XINCREF(v);
            break;
        /* ... */
    }

![offset](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/offset.png)

### after set a value

	>>> a = A()
    >>> a.wing = "wingA"
    >>> a.wing
    'wingA'

because wing attribute of instance `a` is set before, `AttributeError` won't be raised

## instance access x

	>>> a.x
	>>> 3

type of `descr` is `int`, it's not a data descriptor(does not have `__get__` or `__set__` attribute), so the `descr` will be returned directly

![instance_normal](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/instance_normal.png)

you are not able to create any other attribute in instance `a` if `A` defined `__slots__`, we will figure out why later

	>>> a.not_exist = 100
	Traceback (most recent call last):
	File "<input>", line 1, in <module>
	AttributeError: 'A' object has no attribute 'not_exist'

# access type

## type access wing

	>>> A.wing
	<member 'wing' of 'A' objects>
    >>> type(A.wing)
    <class 'member_descriptor'>

the procedure of accessing `A.wing` is nearly the same as `a.wing`

![type_desc](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/type_desc.png)

## type access x

	>>> A.x
	3

the procedure of accessing `A.x` is nearly the same as `a.x`

![type_normal](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/type_normal.png)

# difference

## with slot

### how does attributes initialized in the creation of `class A` ?

we can laern about the creation procedure in [type->creation of class](https://github.com/zpoint/CPython-Internals/blob/master/BasicObject/type/type.md#creation-of-class)

`type` object has many fields in it's C structure, the following picture only shows what's necessary for this topic

what's in the `__slots__` is sorted and stored as a tuple in `ht_slots`

the two attributes in `__slots__` is preallocated in the tail of the newly created type object `A` and two `PyMemberDef` pointers are stored in the order of `ht_slots`

while attribute `x` stays in the `__dict__`

![type_create](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/type_create.png)

### how does attributes initialized in the creation of `instance a` ?

the memory location of the attributes in `__slots__` are preallocated

![instance_create](https://github.com/zpoint/CPython-Internals/blob/master/Interpreter/slot/instance_create.png)

### lookup procedure in MRO ?

## without slot

### how does attributes initialized in the creation of `class A` ?

### how does attributes initialized in the creation of `instance a` ?

### lookup procedure in MRO ?

## memory saving measurement

# read more
* [`__slots__`magic](http://book.pythontips.com/en/latest/__slots__magic.html)
* [A quick dive into Python’s `__slots__`](https://blog.usejournal.com/a-quick-dive-into-pythons-slots-72cdc2d334e?gi=26f20645d6c9)