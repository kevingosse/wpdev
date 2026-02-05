---
title: "Uncovering a bug in Attribute.GetHashCode"
date: 2017-03-26
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
---

It started from [a question on StackOverflow](http://stackoverflow.com/questions/43028703/reflection-renders-hashcode-unstable/). The following code will return inconsistent value for the hashcode of the attribute:

```csharp
static void Main(string[] args)
{
    typeof(SomeClass).GetCustomAttributes(false);//without this line, GetHashCode behaves as expected

    SomeAttribute tt = new SomeAttribute();
    Console.WriteLine(tt.GetHashCode());//Prints 1234567
    Console.WriteLine(tt.GetHashCode());//Prints 0
    Console.WriteLine(tt.GetHashCode());//Prints 0
}

[SomeAttribute(field2 = 1)]
class SomeClass
{
}

class SomeAttribute : System.Attribute
{
    uint field1=1234567;
    public uint field2;
}
```

This will print the output:

> 1234567 0 0

We can see that the hashcode value changes between the first and second invocation. Even more interesting, commenting line 3 seems to fix the behavior and **GetHashCode** will always return 1234567.

To understand what's going on, we first need to look at the source code of the **Attribute.GetHashCode** method:

```csharp
public override int GetHashCode()
{
    FieldInfo[] fields = this.GetType().GetFields(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);

    foreach (FieldInfo field in fields)
    {
        object value = field.GetValue(this);

        if (value != null && !value.GetType().IsArray)
            return value.GetHashCode();
    }

    return this.GetType().GetHashCode();
}
```

In a nutshell, what it does is:

1. Enumerate the fields of your attribute
2. Find the first field that isn't an array and hasn't a null value
3. Return the hashcode of this field

We can make two conclusions at that point:

1. Only one field is taken into account to compute the hashcode of the attribute
2. The algorithm relies heavily on the order of the fields returned by **Type.GetFields** (since we take the first field that matches the conditions)

With those conclusion, we can hypothesize that for some reason, the order of the fields returned by **Type.GetFields** changes over time. This can actually be verified easily:

```csharp
static void Main(string[] args)
{
    void DisplayAttributes()
    {
        var fields = new SomeAttribute()
          .GetType()
          .GetFields(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);

        foreach (var field in fields)
        {
            Console.WriteLine(field.Name);
        }
    }

    typeof(SomeClass).GetCustomAttributes(false);

    DisplayAttributes();
    DisplayAttributes();
}
```

This code will display:

> field1 field2 field2 field1

It confirms that the order of the fields changes. Whenever **field1** comes first, the hashcode will be 1234567. On the other hand, when **field2** comes first, the hashcode will be 0.

Can we find out why the order is changing?

 

# The RuntimeType cache

This apparently has something to do with the **typeof(SomeClass).GetCustomAttributes(false)** line, since everything behaves consistently if we remove it. Poking around in the source code of the **RuntimeType.GetFields** method, we can see that it uses some kind of cache internally. The contents of this cache can be checked with the debugger by poking around in the quickwatch window:

 

[![image](images/image_thumb.png "image")](http://blog.wpdev.fr/wp-content/uploads/2017/03/image.png)

 

The interesting part is the **m\_fieldInfoCache.m\_allMembers** field. This is the actual cache that is used by the **RuntimeType.GetFields** method.

From there, we can experiment a bit. If we call **GetHashCode** first thing in the program, the cache will contain (remember that order is important):

> field1 field2

Therefore, the hashcode will be 1234567.

On the other hand, if we call **typeof(SomeClass).GetCustomAttributes(false)** first, the cache will contain:

> field2

Why field2? Because we access that field on the attribute applied to **SomeClass**:

```csharp
[SomeAttribute(field2 = 1)]
class SomeClass
{
}
```

Then, after calling **GetHashCode**, the cache will be filled out with the missing field and will contain:

> field2 field1

Therefore, subsequent calls to GetHashCode will return 0.

The only mystery left is: in that case, why is the first call to GetHashCode returning 1234657, and the next ones 0? We already know why the next ones return 0, so something special must happen the first time.

The answer lies in the way the cache is implemented. First, in the **RuntimeTypeCache.MemberInfoCache<T>.GetMemberList** method:

```csharp
private T[] GetMemberList(RuntimeType.MemberListType listType, RuntimeType.RuntimeTypeCache.CacheType cacheType)
{
    if (Volatile.Read(ref this.m_cacheComplete))
        return this.m_allMembers;

    return this.Populate(null, listType, cacheType);
}
```

First we check if the cache is complete. If so, we return it directly. If not, we return the result of the **Populate** method. What is this method doing?

```csharp
private unsafe T[] Populate(string name, RuntimeType.MemberListType listType, RuntimeType.RuntimeTypeCache.CacheType cacheType)
{
    T[] listByName;
    if (name == null || name.Length == 0 || cacheType == RuntimeType.RuntimeTypeCache.CacheType.Constructor &&
        (int) name.FirstChar != 46 && (int) name.FirstChar != 42)
    {
        listByName = this.GetListByName((char*) null, 0, (byte*) null, 0, listType, cacheType);
    }
    else
    {
        // ... other logic
    }
    this.Insert(ref listByName, name, listType);
    return listByName;
}
```

(stripped for simplicity) First, we retrieve the list of all fields. Then, the **Insert** method adds to the cache the fields that were previously missing. Finally, we return **the original list**. That’s the keypoint.

**Summing it up:**

- Internally, fields are retrieved in the order “field1, field2”
- When calling **typeof(SomeClass).GetCustomAttributes(false)**, “field2” is put into the cache
- When calling **GetHashCode** the first time, the fields are enumerated. Since the cache is incomplete, the full list of fields is retrieved (in the order “field1, field2”), the missing field is added to the cache (which now contains “field2, field1”), and the original list (“field1, field2”) is returned
- When subsequently calling **GetHashCode**, the fields are enumerated. Since the cache is complete, its value is directly returned, in the order “field2, field1”

 

# Conclusion

I believe this is a bug. The value returned by **GetHashCode** shouldn’t change if the underlying object hasn’t been modified. Otherwise, it’ll cause inconsistent and dangerous behavior with the collections that use it. Consider for instance the following code:

```csharp
var hashSet = new HashSet<SomeAttribute>();

typeof(SomeClass).GetCustomAttributes(false);
SomeAttribute tt = new SomeAttribute();

hashSet.Add(tt);
Console.WriteLine(hashSet.Contains(tt)); // Will display false

hashSet.Add(tt);
Console.WriteLine(hashSet.Contains(tt)); // Will display true

Console.WriteLine(hashSet.Count); // Will display 2
```

We're adding the same instance twice to a hashset. Everytime, we call **Contains** and display the result. The first time, “False” will be displayed. The second time, “True” will be displayed. Furthermore, the hashset reports that it contains two different items even though we added the same one everytime!

All in one, **Attribute.GetHashCode** shouldn’t rely on the order of the fields returned by **Type.GetFields**, as it can change over time.
