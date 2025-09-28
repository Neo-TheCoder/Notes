# write API in unistd.h
The function call stack is displayed as follows:
```c
int write(int fildes, const char * buf, off_t count);

_syscall3(int,write,int,fd,const char *,buf,off_t,count)

#define _syscall3(type,name,atype,a,btype,b,ctype,c) \
type name(atype a,btype b,ctype c) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name),"b" ((long)(a)),"c" ((long)(b)),"d" ((long)(c))); \
if (__res>=0) \
	return (type) __res; \
errno=-__res; \
return -1; \
}

// equivalent functional form by AI
int write(int fd, char* buf, int count)
{
    long __res;
    // asm
    __asm__ volatile ("int $0x80"
        : "=a" (__res) 
        : "0" (__NR_write), "b" ((long)(fd)), "c" ((long)(buf)), "d" ((long)(count)));  // 执行int 0x80则陷入内核态
    if (__res >= 0)
        return (int) __res;
    errno = -__res;
    return -1;
}
```





































