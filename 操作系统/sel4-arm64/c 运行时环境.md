初始参数传递：

```c 
// sel4runtime/src/crt1.c
/*
* This function is simply passed a pointer to the inital stack from the
* C runtime entrypoint.
*
* The stack has the following structure:
*
* * argument count,
* * array of argument pointers,
* * an empty string,
* * array of environment pointers,
* * a null terminator,
* * array of auxiliary vector entries,
* * an 'zero' auxiliary vector, then
* * unspecified data.
*/
```

线程创建时，参数构造：
```c
// libsel4/utils/src/process.c
sel4utils_spawn_process_v();
```