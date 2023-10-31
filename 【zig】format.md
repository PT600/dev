
## format hex
```zig
const std = @import("std");

pub fn main() void {
    // Tested with Zig v0.10.1

    const foo: u8 = 26;
    std.debug.print("0x{x}\n", .{foo}); // "0x1a"
    std.debug.print("0x{X}\n", .{foo}); // "0x1A"

    const bar: u16 = 1;
    std.debug.print("0x{x}\n", .{bar}); //     "0x1"
    std.debug.print("0x{x:2}\n", .{bar}); //   "0x 1"
    std.debug.print("0x{x:4}\n", .{bar}); //   "0x   1"
    std.debug.print("0x{x:0>4}\n", .{bar}); // "0x0001"

    const baz: u16 = 43;
    std.debug.print("0x{x:0>4}\n", .{baz}); // "0x002b"
    std.debug.print("0x{X:0>8}\n", .{baz}); // "0x0000002B"

    const qux: u32 = 0x1a2b3c;
    std.debug.print("0x{X:0>2}\n", .{qux}); // "0x1A2B3C" (not cut off)
    std.debug.print("0x{x:0>8}\n", .{qux}); // "0x001a2b3c"
}

```

 Renders fmt string with args, calling `writer` with slices of bytes.
 If `writer` returns an error, the error is returned from `format` and
 `writer` is not called again.

 The format string must be comptime-known and may contain placeholders following
 this format:
 `{[argument][specifier]:[fill][alignment][width].[precision]}`

 Above, each word including its surrounding [ and ] is a parameter which you have to replace with something:

 - *argument* is either the numeric index or the field name of the argument that should be inserted
   - when using a field name, you are required to enclose the field name (an identifier) in square
     brackets, e.g. {[score]...} as opposed to the numeric index form which can be written e.g. {2...}
 - *specifier* is a type-dependent formatting option that determines how a type should formatted (see below)
 - *fill* is a single character which is used to pad the formatted text
 - *alignment* is one of the three characters `<`, `^`, or `>` to make the text left-, center-, or right-aligned, respectively
 - *width* is the total width of the field in characters
 - *precision* specifies how many decimals a formatted number should have

 Note that most of the parameters are optional and may be omitted. Also you can leave out separators like `:` and `.` when
 all parameters after the separator are omitted.
 Only exception is the *fill* parameter. If *fill* is required, one has to specify *alignment* as well, as otherwise
 the digits after `:` is interpreted as *width*, not *fill*.

 The *specifier* has several options for types:
 - `x` and `X`: output numeric value in hexadecimal notation
 - `s`:
   - for pointer-to-many and C pointers of u8, print as a C-string using zero-termination
   - for slices of u8, print the entire slice as a string without zero-termination
 - `e`: output floating point value in scientific notation
 - `d`: output numeric value in decimal notation
 - `b`: output integer value in binary notation
 - `o`: output integer value in octal notation
 - `c`: output integer as an ASCII character. Integer type must have 8 bits at max.
 - `u`: output integer as an UTF-8 sequence. Integer type must have 21 bits at max.
 - `?`: output optional value as either the unwrapped value, or `null`; may be followed by a format specifier for the underlying value.
 - `!`: output error union value as either the unwrapped value, or the formatted error value; may be followed by a format specifier for the underlying value.
 - `*`: output the address of the value instead of the value itself.
 - `any`: output a value of any type using its default format.

 If a formatted user type contains a function of the type
 ```
 pub fn format(value: ?, comptime fmt: []const u8, options: std.fmt.FormatOptions, writer: anytype) !void
 ```
 with `?` being the type formatted, this function will be called instead of the default implementation.
 This allows user types to be formatted in a logical manner instead of dumping all fields of the type.

 A user type may be a `struct`, `vector`, `union` or `enum` type.

 To print literal curly braces, escape them by writing them twice, e.g. `{{` or `}}`.

## ref
* [source code](https://github.com/ziglang/zig/blob/master/lib/std/fmt.zig)