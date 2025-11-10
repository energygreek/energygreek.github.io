---
title: python arguement types
date: 2021-06-18 10:44:05
tags: [python]
---

```python
# accept tupe and list
def calc(numbers):
    sum = 0
    for n in numbers:
        sum = sum + n
    print(sum)

calc([1,2])
calc((1,2,3))

# accept varidict argument
def calc2(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n
    return sum

calc2(0)
calc2(1,2,3,4)

numbers = [1,2,3]
# transform list to varidcit argument
print(calc(*numbers))


# varidict named arguments will be transformed to dict

# only accept named varidict argument
def person2(**kw):
    if 'hello' in kw:
        passs

    print(kw)

# accept compose argument in sequence: normal argument & named varidict argument
def person(name, age, **kw):
    print('name', name, 'age', age, 'other', kw)

person('mike',12)
person('mike',12, city='sh')
person('mike',12, city='sh', birth=1990)
person2(name='mike',age = 12, city='sh', birth=1990)

# compose normal type argument, varidict type argument, and varidict named type argument 
def f2(a, b, c=0, *d, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'd =', d, 'kw =', kw)

cs=[3,4]
f2(1,2,*cs,hello='hello')


# use * to sepcify named type argument
def f3(a,b,c,*,d,e):
    print('a =', a, 'b =', b, 'c =', c, 'd =', d, 'e =', e)

# don't accept other named argument
f3(1,2,3,d=4,e=5)

# use tupe and dict can call all kind of function
# so *arg and **kw are always used to forward all argument to another function of any type of argument
args = (1,2,3)
kw = {'c':1,'d':2}
f2(*arg, **kw)

# at last, use base type for default value
```
