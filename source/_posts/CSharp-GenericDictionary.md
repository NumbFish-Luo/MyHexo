---
title: C#实现“泛型字典”
date: 2021-10-1 10:30:00
toc: true
tags:
- CSharp
categories:
- CSharp
banner_img: /img/Dictionary.jpg
banner_img_set: /img/Dictionary.jpg
---

曾经遇到的面试题，要求实现一个“泛型字典”，可以**同时存储多种类型**，且对于**值类型要避免装箱**。

设计思路：

1. C#和C/C++不一样，数据类型分为了两种，一种是值类型，一种是引用类型。
2. 字典是一个模板类，本身为引用类型。对于```Dictionary<Key, Value>```，如果Value是一个值类型，那么Value数据不会被装箱，例如：```Dictionary<Key, int>```；而如果Value是引用类型...本来就是引用类型了，所以不存在装箱。
3. 对于此题，初看可能会写出这样的设计：```Dictionary<Key, object>```，即所有数据都统一转成object。虽然同时存储多种类型的功能实现了，但是对于值类型没有避免装箱。
4. 对上述第3条进行改进，结合上述第2条，如果```Dictionary<Key, object>```的object也是Dictionary呢，且这个Dictionary的Value模板参数不是object（防止再套一层...）。对于这样的字典套字典的结构，我的设计如下：

```csharp
public class GenericDictionary<KEY> {
    private Dictionary<string, object> _instance = new Dictionary<string, object>();
    public void Add<VALUE>(KEY key, VALUE value) {
        string typeName = typeof(VALUE).Name;
        object outObject;
        if (!_instance.TryGetValue(typeName, out outObject)) {
            _instance.Add(typeName, new Dictionary<KEY, VALUE>());
            outObject = _instance[typeName];
        }
        Dictionary<KEY, VALUE> dictionary = (Dictionary<KEY, VALUE>)outObject;
        dictionary.Add(key, value);
    }
    public bool TryGetValue<VALUE>(KEY key, out VALUE value) {
        object outObject;
        if (!_instance.TryGetValue(typeof(VALUE).Name, out outObject)) {
            value = default;
            return false;
        }
        Dictionary<KEY, VALUE> dictionary = (Dictionary<KEY, VALUE>)outObject;
        if (!dictionary.TryGetValue(key, out value)) {
            return false;
        }
        return true;
    }
}

// usage:
public class TestObject {
    public int value = 123;
}

GenericDictionary<int> gd = new GenericDictionary<int>();

gd.Add(1, 123);

TestObject to = new TestObject();
to.value = 777;
gd.Add(6, to);

int intOut;
if (gd.TryGetValue(1, out intOut)) {
    Debug.Log(intOut); // 123
} else {
    Debug.Log("Unable to get the int value");
}

TestObject toOut;
if (gd.TryGetValue(6, out toOut)) {
    Debug.Log(toOut.value); // 777
} else {
    Debug.Log("Unable to get the TestObject value");
}
```
