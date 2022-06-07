# 第五章

> 只记录部分内容和个人评论，不能替代书本内容

比较reference和box<T>
引用只是地址，但是需要保证引用never outlive the referents.
rust有一些规则

why use reference?

```rust
let table = hashmap<String, Vec<String>>;
show(table); 
```
对于C++来说，默认Copy语义，对于Rust默认Move语义。

“Because of move semantics, we’ve completely destroyed the entire structure sim‐
ply by trying to print it out. Thanks, Rust!“

References come in two kinds:
* &T(A shared reference),read but not modify its referent, many shared references to a particular value at a time.Shared references are ***Copy***.
* &mut T,Mutable references are ***not Copy***.

Rule:a multiple readers or single writer rule at compile time

!!!! Whereas our original outer for loop took ownership of the
HashMap and consumed it, in our new version it receives a
shared reference to the HashMap. Iterating over a shared
reference to a HashMap is defined to produce shared
references to each entry’s key and value: artist has
changed from a String to a &String, and works from a
Vec<String> to a &Vec<String>.

```rust
show(&table);
```

```rust
fn show(table: &Table) {
    for (artist, works) in table {
        println!("works by {}:", artist);
        for work in works {
            println!(" {}", work);
        }
    }
}

```
* ***by reference***: If we instead pass the function a reference to the value.
* ***by value***: When we pass a value to a function in a way that moves ownership of the value to the function.


## Rust Reference VS C++ Reference

In C++, references are created implicitly by conversion, and dereferenced implicitly.

In Rust, references are created explicitly with the & operator, and dereferenced explic‐
itly with the * operator.

### Rust .operator deference
```rust
// 1
assert_eq!((*anime_ref).name, "Aria: The Animation");
```

```rust
// 2
let mut v = vec![1973, 1968];
v.sort();
(&mut v).sort();
```
In a nutshell, whereas C++ converts implicitly between references and lvalues (that is,
expressions referring to locations in memory), with these conversions appearing any‐
where they’re needed, in Rust you use the & and * operators to create and follow ref‐
erences, with the exception of the . operator, which borrows and dereferences
implicitly.

The println! macro used in the show function expands to
code that uses the . operator, so it takes advantage of this
implicit dereference as well.

??? what println! expands and use .operator?

```rust
struct Anime { name: &'static str, bechdel_pass: bool };
let aria = Anime { name: "Aria: The Animation", bechdel_pass: true };
let anime_ref = &aria;
assert_eq!(anime_ref.name, "Aria: The Animation");
assert_eq!((*anime_ref).name, "Aria: The Animation");

let mut v = vec![1973, 1968];
v.sort(); // implicitly borrows a mutable reference to v
(&mut v).sort(); // equivalent, but more verbose

struct Point { x: i32, y: i32 }
let point = Point { x: 1000, y: 729 };
let r: &Point = &point;
let rr: &&Point = &r;
let rrr: &&&Point = &rr;
assert_eq!(rrr.y, 729);
```

## Assigning References

```rust
let x = 10;
let y = 20;
let mut r = &x; //指向不可变引用的可变引用
if b { r = &y; }
assert!(*r == 10 || *r == 20);
```
## Comparing References

```rust
let x = 10;
let y = 10;
let rx = &x;
let ry = &y;
let rrx = &rx;
let rry = &ry;
assert!(rrx <= rry);
assert!(rrx == rry);

assert!(rx == ry); // their referents are equal
assert!(!std::ptr::eq(rx, ry)); // but occupy different addresses
```

## References Are Never Null

## Borrowing References to Arbitrary Expressions

## References to Slices and Trait Objects

two kinds of fat pointers:
* Reference to a slice: carrying the starting address of the slice and its length.
* a reference to a value that implements a certain trait.A trait object carries a value’s address and a pointer to the trait’s implementation appropriate to that value, for invoking the trait’s methods.

## Reference Safety

### Borrowing a Local Variable

> reference(&) 其实是一个无主指针

***Rule one***: 租借方生命周期不大于出借方

***Lifetime*** Rust编译器使用其用来在编译期borrow check，生命周期从变量初始化开始到no longer use 结束( extending from the point at which
they’re initialized until the point that the compiler can
prove they are no longer in use)。

Rust tries to assign each reference type in your program a lifetime that meets the constraints imposed by how it is used. 

生命周期标注并不改变任何引用的生命周期的长短。与当函数签名中指定了泛型类型参数后就可以接受任何类型一样，当指定了泛型生命周期后函数也能接受任何生命周期的引用。生命周期标注描述了多个引用生命周期相互的关系，而不影响其生命周期。

单个生命周期标注本身没有多少意义，因为生命周期标注告诉 Rust 多个引用的泛型生命周期参数如何相互联系的。

```rust
{
    let r;
    {
        let x = 1;
        r = &x;
    }
    assert_eq!(*r, 1); // bad: reads memory `x` used to occupy
}
```

三个lifetime:r,x,&x

***Constraint one*** the variable’s
lifetime must contain or enclose that of the reference
borrowed from it.避免&x是dangling pointer.
If you have a variable x, then a reference to x must not outlive x itself

***Constraint two*** : 避免r是dangling pointer.
if you store a reference in a variable r, the reference’s type must be good for the entire lifetime of the variable, from its initialization until its last use

The first kind of constraint limits how large a reference’s lifetime can be, while the second kind limits how small it can be.

> Rust simply tries to find a lifetime for each reference that satisfies all these constraints.

一句话：***出借方enclose借用方***.

```rust
{
    let r;
    {
        let x = 1;
        r = &x;
    }
    assert_eq!(*r, 1); // bad: reads memory `x` used to occupy
}

// modified: ver 2
{
    let r;
    {
        let x = 1;
        r = &x;
        assert_eq!(*r, 1); 
    }
}

// modified: ver3
{
    let x = 1;
    {
        let r = &x;
        assert_eq!(*r, 1);
    }
}
```

#### Structs Containing References

编译器可以自动识别函数内部的局部变量的生命周期并进行check,但是当跨词法作用域的borrow，编译器无法进行check，需要手动传入lifetime parameters.

if you store a reference in some data structure, its lifetime must enclose that of the data structure.
For example, if you build a vector of references, all of them must have lifetimes enclosing that of the variable that owns the vector.

```rust
// ??? rust为什么不会隐式给r一个'a?
// 为了reference可见性
// p188
struct S {
    r: &i32
}

let s;
{
    let x = 10;
    s = S { r: &x };
}
assert_eq!(*s.r, 10); // bad: reads from dropped `x`
```

```rust
struct S<'a> {
    r: &'a i32
}
```

```rust
struct D {
    s: S // not adequate
}

struct D {
    s: S<'static>
}

struct D<'a> {
    s: S<'a>
}
```
Now the S type has a lifetime, just as reference types do.
Each value you create of type S gets a fresh lifetime 'a,
which becomes constrained by how you use the value. The
lifetime of any reference you store in r had better enclose
'a, and 'a must outlast the lifetime of wherever you store
the S.

Without looking into the definition of the Record type at all,
we can tell that, if we receive a Record from parse_record,
whatever references it contains must point into the input
buffer we passed in, and nowhere else (except perhaps at
'static values).
In fact, this exposure of internal behavior is the reason
Rust requires types that contain references to take explicit
lifetime parameters. There’s no reason Rust couldn’t simply
make up a distinct lifetime for each reference in the struct
and save you the trouble of writing them out. Early
versions of Rust actually behaved this way, but developers
found it confusing: it is helpful to know when one value
borrows something from another value, especially when
working through errors.


### Lifetime Parameters 

Receiving References as Function Arguments

In other
words, we were unable to write a function that stashed a
reference in a global variable without reflecting that
intention in the function’s signature. In Rust, a function’s
signature always exposes the body’s behavior.
Conversely, if we do see a function with a signature like
g(p: &i32) (or with the lifetimes written out, g<'a>(p: &'a
i32)), we can tell that it does not stash its argument p
anywhere that will outlive the call. There’s no need to look
into g’s definition; the signature alone tells us what g can
and can’t do with its argument. This fact ends up being
very useful when you’re trying to establish the safety of a
call to the function.

### Returning References

```rust

fn f(p: &'static i32) { ... }
let x = 10;
f(&x); // error

// v should have at least one element.
// fn smallest<'a>(v: &'a [i32]) -> &'a i32 { ... }
fn smallest(v: &[i32]) -> &i32 {
    let mut s = &v[0];
    for r in &v[1..] {
    if *r < *s { s = r; }
}
    s
}

let s;
{
    let parabola = [9, 4, 1, 0, 1, 4, 9];
    s = smallest(&parabola);
}
assert_eq!(*s, 0); // bad: points to element of dropped array
```

编译器需要给&parabola一个lifetime，首先看这个lifetime需要满足的constraints.

首先&parabola 的lifetime不能超过parabola(Rule one)，其次smallest(&parabola)的返回引用lifetime需要不低于s的lifetime。
'a是smallest的返回值，也是函数参数，&parabola需要实例化‘a，但是不存在这个lifetime满足上述constraints。


From smallest’s signature, we can see that its argument
and return value must have the same lifetime, 'a. In our
call, the argument &parabola must not outlive parabola itself,
yet smallest’s return value must live at least as long as s.
There’s no possible lifetime 'a that can satisfy both
constraints.

Lifetimes in function signatures let Rust assess the
relationships between the references you pass to the
function and those the function returns, and they ensure
they’re being used safely.


### Distinct Lifetime Parameters
```rust
struct S<'a> {
 x: &'a i32,
 y: &'a i32
}

let x = 10;
let r;
{
    let y = 20;
    {
        let s = S { x: &x, y: &y };
        r = s.x;
    }
}
println!("{}", r); // error
```

```rust

struct S<'a, 'b> {
    x: &'a i32,
    y: &'b i32
}
```


* Both fields of S are references with the same lifetime 'a, so Rust must find a sin‐
gle lifetime that works for both s.x and s.y.

* We assign r = s.x, requiring 'a to enclose r’s lifetime

* We initialized s.y with &y, requiring 'a to be no longer than y’s lifetime.

```rust
struct S<'a, 'b> {
    x: &'a i32,
    y: &'b i32
}

// FIXME: test?
let x = 10;
let r;
{
    let y = 20;
    {
        let s = S { x: &x, y: &y };
        r = s.x;
    }
}
println!("{}", r);
```

### Omitting Lifetime Parameters

* Rule one
```rust
// x omit struct lifetime parameters
struct S<'a, 'b> {
    x: &'a i32,
    y: &'b i32
}

// fn sum_r_xy<'a, 'b, 'c>(r: &'a i32, s: S<'b, 'c>) -> i32
fn sum_r_xy(r: &i32, s: S) -> i32 {
    r + s.x + s.y
}
```

* Rule two
```rust
// fn first_third<'a>(point: &'a [i32; 3]) -> (&'a i32, &'a i32)
fn first_third(point: &[i32; 3]) -> (&i32, &i32) {
    (&point[0], &point[2])
}
```

* Rule three
```rust
// &self、&mut self
struct StringTable {
    elements: Vec<String>,
}

impl StringTable {
    // fn find_by_prefix<'a, 'b>(&'a self, prefix: &'b str) -> Option<&'a String>
    fn find_by_prefix(&self, prefix: &str) -> Option<&String> {
        for i in 0 .. self.elements.len() {
            if self.elements[i].starts_with(prefix) {
                return Some(&self.elements[i]);
            }
        }   
    None
    }
}
```

补充：
http://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/second-edition/ch19-02-advanced-lifetimes.html

### Sharing Versus Mutation

// TODO:

```rust
let v = vec![4, 8, 19, 27, 34, 10];
let r = &v;
let aside = v; // move vector to aside
r[0]; // bad: uses `v`, which is now uninitialized
```
Although v stays in scope for r’s entire lifetime, the problem here is that v’s value gets
moved elsewhere, leaving v uninitialized while r still refers to it. 

Throughout its lifetime, a shared reference makes its referent read-only: you may not
assign to the referent or move its value elsewhere.

```rust
let v = vec![4, 8, 19, 27, 34, 10];
{
    let r = &v;
    r[0]; // ok: vector is still there
}
let aside = v;
```

```rust
fn extend(vec: &mut Vec<f64>, slice: &[f64]) {
        for elt in slice {
            vec.push(*elt);
    }
}

extend(&mut wave, &wave);
```

## Rust’s Shared References Versus C’s Pointers to const

This sort of problem isn’t unique to Rust: modifying collections while pointing into
them is delicate territory in many languages. In C++, the std::vector specification
cautions you that “***reallocation*** [of the vector’s buffer] invalidates all the references,
pointers, and iterators referring to the elements in the sequence.” Similarly, Java says,
of modifying a java.util.Hashtable object:

If the Hashtable is structurally modified at any time after the iterator is created, in any
way except through the iterator’s own remove method, the iterator will throw a
ConcurrentModificationException.

```cpp
auto vec = std::vector<int>{0};
auto begin = vec.begin();
auto end = vec.end();

vec.insert({4,5,6,7,8,9,10});

std::cout << *begin << std::endl;
std::cout << *(begin + 2) << std::endl;
```

* Shared access is read-only access.
* Mutable access is exclusive access.