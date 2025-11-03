The `getifaddrs()` function is a standard Unix function used to retrieve information about network interfaces, including their addresses. It creates a linked list of `ifaddrs` structures, each describing a network interface, and stores the address of the first item in the list. 

However, `getifaddrs()` is ***not part of the POSIX standard***. While it's widely available on Unix-like systems, including Linux (where it first appeared in glibc 2.3), it is not specified by IEEE for maintaining compatibility among operating systems in the same way that core POSIX APIs are.

## 相关Blog
[Is there a POSIX-compliant way of getting local network IP address of my computer?](https://stackoverflow.com/questions/8645566/is-there-a-posix-compliant-way-of-getting-local-network-ip-address-of-my-compute)
>POSIX没有提供标准接口获取本地interfaces；但是有其他手段可以获得 - [参见](https://stackoverflow.com/a/8645922/23558697)


