# Zig Erasure Coding -- WTF is Zig Comptime 2 by Example (Part 1)

The power and complexity of **Comptime** in Zig

---

Ed Yu ([@edyu](https://github.com/edyu) on Github and
[@edyu](https://twitter.com/edyu) on Twitter)
January.17.2024

---

![Zig Logo](https://ziglang.org/zig-logo-dark.svg)

## Introduction

[**Zig**](https://ziglang.org) is a modern systems programming language and although it claims to a be a **better C**, many people who initially didn't need systems programming were attracted to it due to the simplicity of its syntax compared to alternatives such as **C++** or **Rust**.

However, due to the power of the language, some of the syntaxes are not obvious for those first coming into the language. I was actually one such person.

Today we will explore a unique aspect of metaprogramming in **Zig** that sets it apart from other similarly low-level languages -- `comptime`.
I've already written an [earlier article](https://zig.news/edyu/wtf-is-zig-comptime-and-inline-257b) on `comptime` but today I want to explore an example using a simplified version of [erasure coding](https://github.com/edyu/zig-erasure) that I've implemented using **Zig** with extensive use of `comptime`.

Due to the length of the code example, I'll explain the code in a series of articles and eventually even compare it to an [Odin](https://odin-lang.org) implementation.

## WTF is Comptime

I gave an overview of **Zig** `comptime` in [WTF Zig Comptime](https://zig.news/edyu/wtf-is-zig-comptime-and-inline-257b).
In short, `comptime` is a way to designate either a variable or a piece of code such as a block or a function as something to be run at compile time as opposed to runtime.
The first benefit of running a these `comptime` code at `comptime` is so that the results of `comptime` are essentially constants that can be used directly by the compiled program.
As a result, `comptime` do not use runtime resources for these any results. One example is the same `factorial` code I showed in [WTF Zig Comptime](https://zig.news/edyu/wtf-is-zig-comptime-and-inline-257b).
In such a case, even if I pass in a large number `n`, which takes a long time to calculate at `comptime`, during runtime it's just another constant.
By using a `comptime` variable, I still have the flexibility to change the number anytime and just recompile the program to recalculate the correct result.

```zig
    pub fn factorial(comptime n: u8) comptime_int {
        var r = 1; // no need to write it as comptime var r = 1
        for (1..(n + 1)) |i| {
            r *= i;
        }
        return r;
    }

    const v = comptime factorial(120)
```

The other benefit is actually due to necessity of metaprogramming. One of the primary reasons to use metaprogramming is to allow for different behaviors for different types.
Because you have to know precisely how much memory to allocate at compile time, **Zig** types must be complete at compile type.
What it means is that if you want to a *type function* (once again, see [WTF Zig Comptime](https://zig.news/edyu/wtf-is-zig-comptime-and-inline-257b)) to create different types based on some variable you pass, that variable must be comptime.
One example is say you want to create a matrix type and that a matrix type of 3 x 3 should be completely separate from a matrix type of 4 x 4 in that you cannot simply treat them in the same way.
However, you do want to use the same *type function* to create both matrices, then you must declare the `n` in `n x n` matrix as a `comptime` variable that you pass in.

```zig
    // a square matrix is a matrix where number_of_rows == number_of_cols
    pub fn SquareMatrix(comptime n: comptime_int) type {
        return struct {
            allocator: std.mem.Allocator,
            comptime numRows: u8 = n, 
            comptime numCols: u8 = n, 

            const Self = @This(); 

            pub fn init(allocator: std.mem.Allocator) Self {
                return .{
                    .allocator = allocator,
                }
            }

            // other methods
        }
    }
```

## WTF is Erasure Coding 

Now, with the introduction out of the way, let's talk about Erasure Coding.

[Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code) is a way to split up your data in multiple pieces (shards) so that if you lose one or more of the pieces, you can still retrieve the original data.
If you think this sounds like duplication or replication, or even RAID, then you are somewhat correct.
The difference is that in a simple replication (or RAID 1), you make complete copies of the data so that if you make 2 copies of the data, you have a total of 3 copies (including the original copy) and that as long as you don't lose all 3 copies, you can get the data back.
In a more complex RAID setup such as RAID 5, data is split across (stripe) multiple disks and then additional parity blocks are added to tolerate disk loss. However, in the case of RAID 5, you can tolerate 1 disk loss, whereas RAID 6 can tolerate up to 2 disk failures. 

In [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code), you transform the data while you split the data so that the copies are not necessarily the same.
You still need a minimum number of copies to reconstruct the original data and as long as you don't lose more than such minimal number, you can reconstruct the data.

We typically describe [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code) with 2 numbers of N and K where N is the total number of shards, and K is the minimum number of shards you need in order to reconstruct the data.
For instance, in a (5, 3) Erasure Coding, N is 5 and K is 3, you need 3 of the 5 shards in order to reconstruct the data. In other words, you can tolerate the loss of up to (N - K) = 2 shards. 

What makes [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code) better than a simple replication is that you save more disk spaces by using Erasure Coding compared to simple replication. 

In a quick test, using a file of 680 bytes, if I make 2 additional copies, the total storage needed is 680 * 3 = 2040 bytes.
If instead I use my **Zig** [Erasure Coding](https://github.com/edyu/zig-erasure), I need 5 copies of 240 bytes each which means the total storage needed is 240 * 5 = 1020 bytes.
The total saving in storage comparing the two is (2040 - 1020) / 2040 = 50%.

Of course the storage saving is not free as there is extra computation involved both in encoding (splitting the original data to shards) and decoding (reconstruct the data from shards).

## Erasure Code Example

The example [Erasure Coding](https://github.com/edyu/zig-erasure) is based upon [Erasure Coding For The Masses](https://towardsdatascience.com/erasure-coding-for-the-masses-2c23c74bf87e) by [Dr. Vishesh Khemani](https://vishesh-khemani.medium.com/).
The method described is not the most efficient [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code) but it's easy to understand and I really liked how [Dr. Vishesh Khemani](https://vishesh-khemani.medium.com/) used simple [finite field arithmetic](https://en.wikipedia.org/wiki/Finite_field_arithmetic) to describe [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code). 
[finite field arithmetic](https://en.wikipedia.org/wiki/Finite_field_arithmetic) to describe [Erasure Coding](https://en.wikipedia.org/wiki/Erasure_code). 

## Finite (Binary) Field Arithmetic

If you are mathmetically inclined, you are certainly welcome to read the [Wikipedia article](https://en.wikipedia.org/wiki/Finite_field_arithmetic) or in particular [binary fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html).
I'm not but I'll try my best to give a simple explanation of [binary fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html) from a programmer's perspective.

When you have a [finite field](https://en.wikipedia.org/wiki/Finite_field_arithmetic), you are basically doing [modular maths](https://en.wikipedia.org/wiki/Modular_arithmetic), which basically means that you have to take the modulo of the modulus (usually 1 more than then largest allowed number).
For example, for a finite field of 3, you only have `[0, 1, 2]` as valid numbers, the **modulus** is 3 (1 more than the largest number which is 2).
So if you add 1 and 2 together, you normally would get 3 without [modular arithmetic](https://en.wikipedia.org/wiki/Modular_arithmetic) but because it's [modulo maths](https://en.wikipedia.org/wiki/Modular_arithmetic), you have to modulo 3 which is `3 mod 3 = 0`.
For us programmers, this is very similar to how a [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) works.  

In the case of [binary fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html), the maths is even easier as you only have `[0, 1]` as valid numbers and addition is really just `XOR` because `1 + 1 = 2 mod 2 = 0 = 1 XOR 1`. 
What's even cooler is that because `-1 = 1 mod 2` (try going counter-clockwise on your [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer)), we can do subtraction just as we do addition because `n - 1 = n + (-1) = n + 1`.  

## Code: matrix.zig

Ok, that's probably too much maths already. Let's go back to the [code example](https://github.com/edyu/wtf-zig-erasure).

The [first code](https://github.com/edyu/wtf-zig-erasure/matrix.zig) I want to describe is an implementation of `m x n` matrix where `m` is the number of rows and `n` is the number of columns.

I decided to use *type function* to return the matrix type because I wanted to make sure the compiler can enforce that a `3 x 2` matrix is different from a `2 x 3` matrix.

The way to do so is to pass in both `m` and `n` as `comptime` variables to the *type function*. 

```zig
    pub fn Matrix(comptime m: comptime_int, comptime n: comptime_int) type {
        return struct {
            allocator: std.mem.Allocator = undefined,
            mdata: std.ArrayList(u8) = undefined,
            mtype: DataOrder = undefined,
            comptime numRows: u8 = m,
            comptime numCols: u8 = n,

            const Self = @This();

            // followed by type methods
        }
    }
```

Do note that I ended up saving `m` and `n` as fields inside the `Matrix(m, n)` type as `numRows` and `numCols`.
**Zig** doesn't require you to only access fields as methods and by exposing them as fields, I can use `mat.numRow` and `mat.numCols` directly if `mat` is a `Matrix(m, n)`. 
I do have to designate both `numRows` and `numCols` as `comptime` in order to store `m` and `n`.

The code should be fairly straightforward but I do want to explain why I have an `enum` called `DataOrder`.
The internal data format is an array using `ArrayList` called `mdata`. Typically, you can store a 2D matrix as 1D vector (or `ArrayList` in **Zig**) either in row-major or column-major order.
The original reason to have `DataOrder` is so that I can have the flexibility to store the matrix data in either order and then `transpose` is really just changing the `mtype` from one to the other.

In the end, not only I ended up not needing such flexibiltiy, but it made my code much more complicated. I kept it there mostly as a reminder to myself to not do [premature optimization](https://wiki.c2.com/?PrematureOptimization). 

    Premature Optimization is the root of all evil
                                -- Donald Knuth


One artifact of such *complication* is that `getSlice()` returns a vector based on whether the matrix is stored internally in row-major or colume-major.
The burden is on the caller to keep track of of the `DataOrder` because the caller is the one that initialized the matrix.
Of course, `getRow()` and `getCol()` are exposed and preferred.

```zig
        // return eithers a row or a column based on matrix data order
        // use getRow() or getCol() if you want row/col regardless of data order
        pub fn getSlice(self: *const Self, rc: usize) []u8 {
            switch (self.mtype) {
                .row => {
                    std.debug.assert(rc < m);
                    const i = rc * n;
                    return self.mdata.items[i .. i + n];
                },
                .col => {
                    std.debug.assert(rc < n);
                    const i = rc * m;
                    return self.mdata.items[i .. i + m];
                },
            }
        }
```

The reverse `setSlice()` has similar logic and in the end, I'm not sure whether the *optimization* is ever needed.
Once again, it's probably better to not go through such complication and just expose `setRow()` and `setCol()` instead.

```zig
        fn setSlice(self: *Self, rc: usize, new_rc: []const u8) void {
            switch (self.mtype) {
                .row => {
                    std.debug.assert(rc < m and new_rc.len >= n);
                    const i = rc * n;
                    self.mdata.replaceRange(i, n, new_rc[0..n]) catch return;
                },
                .col => {
                    std.debug.assert(rc < n and new_rc.len >= m);
                    const i = rc * m;
                    self.mdata.replaceRange(i, m, new_rc[0..m]) catch return;
                },
            }
        }
```

## Code: finite_field.zig

Finally, we can look at the [code](https://github.com/edyu/wtf-zig-erasure/finite_field.zig) for [binary fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html).
The [finite_field.zig](https://github.com/edyu/wtf-zig-erasure/finite_field.zig) exposes the n of finite field p^n^ as `n` using `comptime`: 
Because we only care about [binary finite fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html), `p` is always 2 and doesn't needed to specifically passed in.

```zig
    pub fn BinaryFiniteField(comptime n: comptime_int) type {
        return struct {
            exp: u8 = undefined,
            order: u8 = undefined,
            divisor: u8 = undefined,
            // more code
        }
    }
```

The `init()` method mostly initializes the 3 numbers needed later.
The `exp` is just the `n` passed in.
Note that because of the use case and my lack of advanced mathematics knowledge, I only care about `n >= 1 and n <= 7`. 
This also made it easier for calculating the `order` because `u8` is enough to left shift 1 based on the `n`. 
The `divisor` are primes that are results of using 2 as `x` in the comments.
Leave in the comments if you know the maths and want to explain to other readers why that's the case.

```zig
        pub fn init() ValueError!Self {
            var d: u8 = undefined;

            // Irreducible polynomial for mod multiplication
            d = switch (n) {
                1 => 3, // 1 + x ? undef  shift(0b11)=2
                2 => 7, // 1 + x + x^2    shift(0b111)=3
                3 => 11, // 1 + x + x^3   shift(0b1011)=4
                4 => 19, // 1 + x + x^4   shift(0b10011)=5
                5 => 37, // 1 + x^2 + x^5 shift(0b100101)=6
                6 => 67, // 1 + x + x^6   shift(0b1000011)=7
                7 => 131, // 1 + x + x^7  shift(0b10000011)=8
                else => return ValueError.InvalidExponentError,
            };

            return .{
                .exp = @intCast(n),
                .divisor = d,
                .order = @as(u8, 1) << @intCast(n),
            };
        }
```

The number `order` is the 1 + the maximum allowed number in the fields.
For example, for `n = 3`, the field cannot contain any number `> 10`. 

```zig
        pub fn validated(self: *const Self, a: usize) ValueError!u8 {
            if (a < self.order) {
                return @intCast(a);
            } else {
                return ValueError.InvalidNumberError;
            }
        }
```

## Addition and Subtraction

Recall that for [binary fields](https://www.doc.ic.ac.uk/~mrh/330tutor/ch04s04.html), addition is just `exclusive OR`. 

```zig
        pub fn add(self: *const Self, a: usize, b: usize) ValueError!u8 {
            return try self.validated((try self.validated(a)) ^ (try self.validated(b)));
        }
```

Also, negation is simply the number itself and subtraction is just addition of the negation.

```zig
        pub fn neg(self: *const Self, a: usize) ValueError!u8 {
            return try self.validated(a);
        }

        pub fn sub(self: *const Self, a: usize, b: usize) ValueError!u8 {
            return try self.add(a, try self.neg(b));
        }
```

## Multiplication

Multiplication is probably the most complicated of all the methods except in the case of `n = 1`. 

```zig
        pub fn mul(self: *const Self, a: usize, b: usize) ValueError!u8 {
            if (self.exp == 1) {
                return self.validated(try self.validated(a) * try self.validated(b));
            }

            // n > 1
            const x = try self.validated(a);
            const y = try self.validated(b);
            var result: u16 = 0;
            for (0..8) |i| {
                const j = 7 - i;
                if (((y >> @intCast(j)) & 1) == 1) {
                    result ^= @as(u16, x) << @intCast(j);
                }
            }
            while (result >= self.order) {
                // count how many binary digits result has
                var j = countBits(result);
                j -= self.exp + 1;
                result ^= @as(u16, self.divisor) << @intCast(j);
            }
            return try self.validated(result);
        }
```
It requires a function to count the number of binary digits of a number.
There is probably a much better way to figure this out in **Zig** but since I don't know, I just implemented a naive solution.
Leave a comment if you know a better way.

```zig
        fn countBits(num: usize) u8 {
            var v = num;
            var c: u8 = 0;
            while (v != 0) {
                v >>= 1;
                c += 1;
            }
            return c;
        }
```

## Division

Division is simply the multiplication of the inverse.

```zig
        pub fn div(self: *const Self, a: usize, b: usize) ValueError!u8 {
            return try self.mul(a, try self.invert(b));
        }
```

However, finding the inverse of a number is more complicated than negation.
We do have to check whether the number is 0 first, hence no inverse.
We then find the inverse using brute force by trying every number within the `order` to see whether the result is 1 because `b` is the inverse of `a` if and only if `a * b == 1`.
Once again, if you have a better way, leave a comment.

```zig
        pub fn invert(self: *const Self, a: usize) ValueError!u8 {
            if (try self.validated(a) == 0) {
                return ValueError.NoInverseError;
            }
            for (0..self.order) |b| {
                if (try self.mul(a, b) == 1) {
                    return try self.validated(b);
                }
            }
            return ValueError.NoInverseError;
        }
```

## Matrix Representation

The final 3 methods are used to find the `n x n` matrix representation for a number in the field.   
This is also the primary reason why we had to implement a [matrix](https://github.com/edyu/wtf-zig-erasure/matrix.zig).
The main idea is to separate the number into its basis based on the polynormials mentioned in the comments of `init()`.

```zig
        fn setCol(m: *mat.Matrix(n, n), c: usize, a: u8) void {
            for (0..n) |r| {
                const v = (a >> @intCast(r)) & 1;
                m.set(r, c, v);
            }
        }

        fn setAllCols(self: *const Self, m: *mat.Matrix(n, n), a: usize) !void {
            var basis: u8 = 1;
            for (0..n) |c| {
                const p = try self.mul(a, basis);
                basis <<= 1;
                setCol(m, c, p);
            }
        }

        // n x n binary matrix representation
        pub fn toMatrix(self: *const Self, allocator: std.mem.Allocator, a: usize) !mat.Matrix(n, n) {
            var m = try mat.Matrix(n, n).init(allocator, mat.DataOrder.row);
            try self.setAllCols(&m, a);
            return m;
        }
```

## Bonus

By adding a function called `format` to [matrix.zig](https://github.com/edyu/wtf-zig-erasure/matrix.zig), it's much easier to print out the matrix in debugging calls such as `std.debug.print` because **Zig** would implicitly check whether such method exists for the `struct` and use it to format the output. 
As such I can just call `std.debug.print("{}", mat)` if `mat` is of type `Matrix(m, n)`.

```zig
        pub fn format(self: *const Self, comptime _: []const u8, _: std.fmt.FormatOptions, stream: anytype) !void {
            switch (self.mtype) {
                .row => {
                    try stream.print("\n{d}x{d} row ->\n", .{ m, n });
                    for (0..m) |r| {
                        for (0..n) |c| {
                            try stream.print("{d} ", .{self.get(r, c)});
                        }
                        try stream.print("\n", .{});
                    }
                },
                .col => {
                    try stream.print("\n{d}x{d} col ->\n", .{ m, n });
                    for (0..n) |c| {
                        for (0..m) |r| {
                            try stream.print("{d} ", .{self.get(r, c)});
                        }
                        try stream.print("\n", .{});
                    }
                },
            }
        }
```

## The End

You can read my part 1 at [WTF Zig Comptime](https://zig.news/edyu/wtf-is-zig-comptime-and-inline-257b).

The inspiration of the code is from [Erasure Coding For The Masses](https://towardsdatascience.com/erasure-coding-for-the-masses-2c23c74bf87e). 

You can find the code for the article [here](https://github.com/edyu/wtf-zig-erasure).

The full erasure coding code is [here](https://github.com/edyu/zig-erasure).

## ![Zig Logo](https://ziglang.org/zero.svg)
