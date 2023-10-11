## Strong vs Weak Memory Ordering
* A CPU with a strong memory model gives some important guarantees that make much of the semantics we use in the abstract machine no-ops. It does nothing fancy at all, and is **only a hint to the compiler to not change the order of memory operations** from what we as programmers wrote.

* On a weak system however, it might need to set up memory fences or use special instructions to prevent synchronization issues.

Most current desktop CPUs from AMD and Intel uses strong ordering. That means that the CPU itself gives some guarantees about not reordering certain operations. Some examples of this guarantee can be found in Intels Developer Manual, chapter 8.2.2:
* Reads are not reordered with other reads.
* Writes are not reordered with older reads.
* Writes to memory are not reordered with other writes (with some exceptions)

Critically, it also includes one non-guarantee:
* Reads may be reordered with older writes to different locations but not with older writes to the same location.

## refer
* [Explaining Atomics in Rust](https://cfsamsonbooks.gitbook.io/explaining-atomics-in-rust/)