# Extension members

## What's changed

This proposal is an update on [Anonymous extension declarations](https://github.com/dotnet/csharplang/blob/main/meetings/working-groups/extensions/anonymous-extension-declarations.md) in the following ways:

- It leans into a parameter-style syntax for specifying the receiver.
- It includes syntax for generating compatible extension methods.
- It clarifies the goals of lowering, while leaving more details to implementation.

### Receiver specification: `this` vs parameter

 The strongest sentiment I've heard is that whichever approach we take should be done *consistently*. Of the two there's a lean towards the parameter approach. This proposal leans into the parameter-based approach as strongly as it can.

### Compatibility with classic extension methods

There is a clear desire to be able to move classic extension methods forward into new syntax without breaking existing callers. This proposal lets extension methods be marked for compatibility, which will cause them to generate visible static methods in the pattern of classic extension methods.

### Lowering

The proposal takes the stance that the declarations generated from lowering - static methods and/or types - should be hidden from the language level, making extension members a "real" abstraction. The implementation strategy needs to establish metadata and rules that can be followed and consumed by other compilers, and that ensure stability and compatibility for consuming code as APIs evolve.

## Declaration

### Static classes as extension containers

Extensions are declared inside top-level non-generic static classes, just like extension methods today, and can thus coexist with classic extension methods and non-extension static members:

``` c#
public static class Enumerable
{
    // New extension declaration
    extension(IEnumerable source) { ... }
    
    // Classic extension method
    public static IEnumerable<TResult> Cast<TResult>(this IEnumerable source) { ... }
    
    // Non-extension member
    public static IEnumerable<int> Range(int start, int count) { ... } 
}
```

### Extension declarations

An extension declaration is anonymous, and provides a _receiver specification_ with any associated type parameters and constraints, followed by a set of extension member declarations. The receiver specification may be in the form of a parameter, or - if only static extension members are declared - a type:

``` c#
public static class Enumerable
{
    extension(IEnumerable source) // extension members for IEnumerable
    {
        public bool IsEmpty { get { ... } }
    }
    extension<TSource>(IEnumerable<TSource> source) // extension members for IEnumerable<TSource>
    {
        public IEnumerable<T> Where(Func<TSource, bool> predicate) { ... }
        public IEnumerable<TResult> Select<TResult>(Func<TSource, TResult> selector) { ... }
    }
    extension<TElement>(IEnumerable<TElement>) // static extension members for IEnumerable<TElement>
        where TElement : INumber<TElement>
    {
        public static IEnumerable<TElement> operator +(IEnumerable<TElement> first, IEnumerable<TElement> second) { ... }
    }
}
```

The type in the receiver specification is referred to as the _receiver type_ and the parameter name, if present, is referred to as the _receiver parameter_.

### Extension members

Extension member declarations are syntactically identical to corresponding instance and static members in class and struct declarations (with the exception of constructors). Instance members refer to the receiver with the receiver parameter name:

``` c#
public static class Enumerable
{
    extension(IEnumerable source)
    {
        // 'source' refers to receiver
        public bool IsEmpty => !source.GetEnumerator().MoveNext();
    }
}
```

It is an error to specify an instance extension member (method, property, indexer or event) if the enclosing extension declaration does not specify a receiver parameter:

``` c#
public static class Enumerable
{
    extension(IEnumerable) // No parameter name
    {
        public bool IsEmpty => true; // Error: instance extension member not allowed
    }
}
```

### Refness

By default the receiver is passed to instance extension members by value, just like other parameters. However, an extension declaration receiver in parameter form can specify `ref`, `ref readonly` and `in`, as long as the receiver type is known to be a value type. 

If `ref` is specified, an instance member or one of its accessors can be declared `readonly`, which prevents it from mutating the receiver:

``` c#
public static class Bits
{
    extension(ref ulong bits) // receiver is passed by ref
    {
        public bool this[int index]
        {
            set => bits = value ? bits | Mask(index) : bits & ~Mask(index); // mutates receiver
            readonly get => (bits & Mask(index)) != 0;                // cannot mutate receiver
        }
    }
    static ulong Mask(int index) => 1ul << index;
}
```

### Nullability and attributes

Receiver types can be or contain nullable reference types, and receiver specifications that are in the form of parameters can specify attributes:

``` c#
public static class NullableExtensions
{
    extension(string? text)
    {
        public string AsNotNull => text is null ? "" : text;
    }
    extension([NotNullWhen(false)] string? text)
    {
        public bool IsNullOrEmpty => text is null or [];
    }
    extension<T> ([NotNull] T t) where T : class?
    {
        public void ThrowIfNull() => ArgumentNullException.ThrowIfNull(t);
    }
}
```

### Compatible extension methods

By default, extension members are lowered in such a way that the generated artifacts are not visible at the language level. However, if the receiver specification is in the form of a parameter and specifies the `this` modifier, then any extension instance methods in that extension declaration will generate visible classic extension methods.

Specifically the generated static method has the attributes, modifiers and name of the declared extension method, as well as type parameter list, parameter list and constraints list concatenated from the extension declaration and the method declaration in that order:

``` c#
public static class Enumerable
{
    extension<TSource>(this IEnumerable<TSource> source) // Generate compatible extension methods
    {
        public IEnumerable<TSource> Where(Func<TSource, bool> predicate) { ... }
        public IEnumerable<TSource> Select<TResult>(Func<TSource, TResult> selector)  { ... }
    }
}
```

Generates:

``` c#
public static class Enumerable
{
    public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate) { ... }
    public static IEnumerable<TSource> Select<TSource, TResult>(this IEnumerable<TSource> source, Func<TSource, TResult> selector)  { ... }
}
```

This does not change anything else about how the declared extension method works. However, adding the `this` modifier may lead to a binary break for consumers because the generated artifact may change.

### Constructors

Constructors are generally described as an instance member in C#, since their body has access to the newly created value through the `this` keyword. This does not work well for the parameter-based approach to instance extension members, though, since there is no prior value to pass in as a parameter.

Instead, extension constructors work more like static factory methods. They are considered static members in the sense that they don't depend on a receiver parameter name. Their bodies need to explicitly create and return the construction result. The member itself is still declared with constructor syntax, but cannot have `this` or `base` initializers and does not rely on the receiver type having accessible constructors.

This also means that extension constructors can be declared for types that have no constructors of their own, such as interfaces and enum types:

``` c#
public static class Enumerable
{
    extension(IEnumerable<int>)
    {
        public static IEnumerable(int start, int count) => Range(start, count);
    }
    public static IEnumerable<int> Range(int start, int count) { ... } 
}
```

Allows:

```
var range = new IEnumerable<int>(1, 100);
```

### Operators

Although extension operators have explicit operand types, they still need to be declared within an extension declaration:

``` c#
public static class Enumerable
{
    extension<TElement>(IEnumerable<TElement>) where TElement : INumber<TElement>
    {
        public static IEnumerable<TElement> operator *(IEnumerable<TElement> vector, TElement scalar) { ... }
        public static IEnumerable<TElement> operator *(TElement scalar, IEnumerable<TElement> vector) { ... }
    }
}
```

This allows type parameters to be declared and inferred, and is analogous to how a regular user-defined operator must be declared within one of its operand types.

## Checking

__Inferrability:__ All the type parameters of an extension declaration must be used in the receiver type. This makes it always possible to infer the type arguments when applied to a receiver of the given receiver type.

__Uniqueness:__ Within a given enclosing static class, the set of extension member declarations with the same receiver type (modulo identity conversion and type parameter name substitution) are treated as a single declaration space similar to the members within a class or struct declaration, and are subject to the same [rules about uniqueness](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/classes.md#153-class-members).

``` c#
public static class MyExtensions
{
    extension<T1>(IEnumerable<int>) // Error! T1 not inferrable
    {
        ...
    }
    extension<T2>(IEnumerable<T2>)
    {
        public bool IsEmpty { get ... }
    }
    extension<T3>(IEnumerable<T3>?)
    {
        public bool IsEmpty { get ... } // Error! Duplicate declaration
    }
}
```

The application of this uniqueness rule includes classic extension methods within the same static class. For the purposes of comparison with methods within extension declarations, the `this` parameter is treated as a receiver specification along with any type parameters mentioned in that receiver type, and the remaining type parameters and method parameters are used for the method signature:

``` c#
public static class Enumerable
{
    public static IEnumerable<TResult> Cast<TResult>(this IEnumerable source) { ... }
    
    extension(IEnumerable source) 
    {
        IEnumerable<TResult> Cast<TResult>() { ... } // Error! Duplicate declaration
    }
}
```

## Consumption

When an extension member lookup is attempted, all extension declarations within static classes that are `using`-imported contribute their members as candidates, regardless of receiver type. Only as part of resolution are candidates with incompatible receiver types discarded. A full generic type inference is attempted between the type of the actual receiver and any type parameters in the declared receiver type.

The inferrability and uniqueness rules mean that the name of the enclosing static type is sufficient to disambiguate between extension members on a given receiver type. As a strawman, consider `E @ T` as a disambiguation syntax meaning on a given expression `E` begin member lookup for an immediately enclosing expression in type `T`. For instance:

``` c#
string[] strings = ...;
var query  = (strings @ Enumerable).Where(s => s.Length > 10);
 
public static class Enumerable
{
    extension<T>(IEnumerable<T>)
    {
        public IEnumerable<T> Where(Func<T, bool> predicate) { ... }
    }
}
```

Means lookup `Where` in the type `Enumerable` with `strings` as its receiver. A type argument `string` can now be inferred for `T` from the type of `strings` using standard generic type inference.

A similar approach also works for types: `T1 @ T2` means on a given type `T1` begin static member lookup for an immediately enclosing expression in type `T2`.

This disambiguation approach should work not only for new extension members but also for classic extension methods.

Note that this is not a proposal for a specific disambiguation syntax; it is only meant to illustrate how the inferrability and uniqueness rules enable disambiguation without having to explicitly specify type arguments for an extension declaration's type parameters.

## Lowering

The lowering strategy for extension declarations is not a language level decision. However, beyond implementing the language semantics it must satisfy certain requirements:

- The format of generated types, members and metadata should be clearly specified in all cases so that other compilers can consume and generate it.
- The generated artifacts should be hidden from the language level not just of the new C# compiler but of any existing compiler that respects CLI rules (e.g. modreq's).
- The generated artifacts should be stable, in the sense that reasonable later modifications should not break consumers who compiled against earlier versions.

These requirements need more refinement as implementation progresses, and may need to be compromised in corner cases in order to allow for a reasonable implementation approach.

## Order of implementation

We do not need to implement all of this design at once, but can approach it one or a few member kinds at a time. Based on known scenarios in our core libraries, we should work in the following order:

1. Properties and methods (instance and static)
2. Operators
3. Indexers (instance and static, may be done opportunistically at an earlier point)
4. Anything else

## Future work

### Disambiguation

We still need to settle on a disambiguation syntax. Per the above it needs to be able to take a receiver expression or type, as well as the name of a static class from which to begin member lookup. However, the feature does not have to be extension-specific. There are several cases in C# where it's awkward to get to the right member of a given receiver. Casting often works, but can lead to boxing that may be too expensive or lead to mutations being lost.

It would probably be unfortunate to ship extension members without a disambiguation syntax, so this has high priority.

### Shorter forms

The proposed design avoids per-member repetition of receiver specifications, but does end up with extension members being nested two-deep in a static class _and_ and extension declaration. It will likely be common for static classes to contain only one extension declaration or for extension declarations to contain only one member, and it seems plausible for us to allow syntactic abbreviation of those cases.

__Merge static class and extension declarations:__

``` c#
public static class EmptyExtensions : extension(IEnumerable source)
{
    public bool IsEmpty => !source.GetEnumerator().MoveNext();
}
```

This ends up looking more like what we've been calling a "type-based" approach, where the container for extension members is itself named.

__Merge extension declaration and extension member:__ 

``` c#
public static class Bits
{
    extension(ref ulong bits) public bool this[int index]
    {
        get => (bits & Mask(index)) != 0;
        set => bits = value ? bits | Mask(index) : bits & ~Mask(index);
    }
    static ulong Mask(int index) => 1ul << index;
}
 
public static class Enumerable
{
    extension<TSource>(IEnumerable<TSource> source) public IEnumerable<TSource> Where(Func<TSource, bool> predicate) { ... }
}
```

This ends up looking more like what we've been calling a "member-based" approach, where each extension member contains its own receiver specification. 

