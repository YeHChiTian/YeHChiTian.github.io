---
title: Linux关于select中FD_相关宏实现
date: 2019-11-11 22:09:05
categories: 
- Linux
tags:
- FD_宏
- 位数组
---



# 前言

> 今天看到select中关于 **FD_CLR,FD_ISSET,FD_SET,FD_ZERO**四个宏定义，顺便看看其内部如何实现。 其实现主要靠的是**位数组和位运算**。 本文主要记录在linux上分析这四个宏实现的源码。
>



# 一、FD_四个宏介绍

通过**select**文档，可以看到几个宏的相关声明（通过**man**命令查询文档）在**sys/select.h**文件，其主要是为select以及pselect两个多路IO服务。 而select使用**fd_set**数据结构管理描述符集合，那么将涉及另外问题，如何将一个描述符添加到**fd_set**集合或者从改集合删除，四个宏**FD_**就是来完成这个任务的。下面简单描述这四个宏。

- FD_CLR宏将描述符fd从集合fd_set中移除
- FD_ISSET宏判断描述符fd是否在集合fd_set
- FD_SET宏将描述符fd添加到集合fd_set
- FD_ZERO宏将清空描述符集合fd_set

```c++
SELECT(2)                           Linux Programmer's Manual                       

NAME
       select, pselect, FD_CLR, FD_ISSET, FD_SET, FD_ZERO - synchronous I/O multiplexing

SYNOPSIS
       /* According to POSIX.1-2001, POSIX.1-2008 */
       #include <sys/select.h>

       /* According to earlier standards */
       #include <sys/time.h>
       #include <sys/types.h>
       #include <unistd.h>

       int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);

       void FD_CLR(int fd, fd_set *set);
       int  FD_ISSET(int fd, fd_set *set);
       void FD_SET(int fd, fd_set *set);
       void FD_ZERO(fd_set *set);

       #include <sys/select.h>

       int pselect(int nfds, fd_set *readfds, fd_set *writefds,
                   fd_set *exceptfds, const struct timespec *timeout,
                   const sigset_t *sigmask);

```

# 二、FD_宏相关实现

在/usr/include/x86_64-linux-gnu/sys/select.h文件中查看FD_宏实现，其又依赖于**__FD\_**相关宏。

```c++
/* Access macros for `fd_set'.  */
#define	FD_SET(fd, fdsetp)	__FD_SET (fd, fdsetp)
#define	FD_CLR(fd, fdsetp)	__FD_CLR (fd, fdsetp)
#define	FD_ISSET(fd, fdsetp)	__FD_ISSET (fd, fdsetp)
#define	FD_ZERO(fdsetp)		__FD_ZERO (fdsetp)
```

关于\_\_FD相关定义，__FD_ZERO宏定义有两个版本（一个是GNU CC标准，一个是非GNU CC），在/usr/include/x86_64-linux-gnu/bits/select.h文件中可查看 _FD定义。

```c++
#define __FD_SET(d, set) \
  ((void) (__FDS_BITS (set)[__FD_ELT (d)] |= __FD_MASK (d)))
#define __FD_CLR(d, set) \
  ((void) (__FDS_BITS (set)[__FD_ELT (d)] &= ~__FD_MASK (d)))
#define __FD_ISSET(d, set) \
  ((__FDS_BITS (set)[__FD_ELT (d)] & __FD_MASK (d)) != 0)
#if defined __GNUC__ && __GNUC__ >= 2

//关于__FD_ZERO宏。 有两个版本的实现
# if __WORDSIZE == 64
#  define __FD_ZERO_STOS "stosq"
# else
#  define __FD_ZERO_STOS "stosl"
# endif

# define __FD_ZERO(fdsp) \
  do {									      \
    int __d0, __d1;							      \
    __asm__ __volatile__ ("cld; rep; " __FD_ZERO_STOS			      \
			  : "=c" (__d0), "=D" (__d1)			      \
			  : "a" (0), "0" (sizeof (fd_set)		      \
					  / sizeof (__fd_mask)),	      \
			    "1" (&__FDS_BITS (fdsp)[0])			      \
			  : "memory");					      \
  } while (0)

#else	/* ! GNU CC */

/* We don't use `memset' because this would require a prototype and
   the array isn't too big.  */
# define __FD_ZERO(set)  \
  do {									      \
    unsigned int __i;							      \
    fd_set *__arr = (set);						      \
    for (__i = 0; __i < sizeof (fd_set) / sizeof (__fd_mask); ++__i)	      \
      __FDS_BITS (__arr)[__i] = 0;					      \
  } while (0)

#endif	/* GNU CC */
```

 \_\_FD宏定义都依赖于 _FDS_BITS。在/usr/include/x86_64-linux-gnu/sys/select.h， 可以看其实现以及**fd_set**结构。

```c++
/* The fd_set member is required to be an array of longs.  */
typedef long int __fd_mask;

/* Some versions of <linux/posix_types.h> define this macros.  */
#undef	__NFDBITS
/* It's easier to assume 8-bit bytes than to get CHAR_BIT.  */
#define __NFDBITS	(8 * (int) sizeof (__fd_mask))
#define	__FD_ELT(d)	((d) / __NFDBITS)
#define	__FD_MASK(d)	((__fd_mask) (1UL << ((d) % __NFDBITS)))

/* fd_set for select and pselect.  */
typedef struct
  {
    /* XPG4.2 requires this member name.  Otherwise avoid the name
       from the global namespace.  */
#ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->fds_bits)
#else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->__fds_bits)
#endif
  } fd_set;
```

从中可以知道: (假如在64位操作系统)

* fd_set集合中包含一个成员fds_bits，是一个数组，元素类型为**__fd_mask**，其实是long int 类型（在64位操作系统中，代表8个字节，32位操作系统4个字节），数组大小为 _FD_SETSIZE / __NFDBITS。因此fd_set结构主要就是为了维护一个数组（本质实现为**位数组**）

* \_\_FD_SETSIZE值是系统默认定义为**1024**，表示最多可以存放1024个描述符（位于文件：/usr/include/x86_64-linux-gnu/bits/typesizes.h）

* \_\_NFDBITS值依赖\_\_fd_mask类型字节，将\_\_fd_mask字节数**\*8**, 那是因为一个字节刚好有8位,\_\_fd\_mask是8个字节（ 使用二进制表示一共有2^64位），\_\_NFDBITS值为64。  

  \_\_FD_SETSIZE/\_\_NFDBITS 即表示为1024/64=16， 因此fd_set结构中成员fd_bits数组大小为16， 而每一个元素可以表示64位，每一位将代表一个描述符， 所以16个元素可以表示1024个描述符。

```c++
#define __NFDBITS	(8 * (int) sizeof (__fd_mask))
```

* \_\_FD_ELT(d), 表示d/\_\_NFDBITS, 即在位数组中第几个元素。 假如描述符d=8, 那么会将d描述符保存在fds_bits数组中0号元素。d=351, 351/64 = 5, 那么保存d描述符到fds_bits数组5号元素。

```C++
#define	__FD_ELT(d)	((d) / __NFDBITS)
```

* __FD_MASK(d), 表示将1左移d%\_\_NFDBITS位, 值记为r，即表示描述符d存在fds_bits数组某个元素中的第r位。假如d=8, r = 8, 表示描述符即将存在fds_bits数组中0号元素中第8位（第0号元素二进制第8位为1），假如d=351, r = 31, 表示描述符即将存在fds_bits数组中5号元素中第31位（第5号元素二进制第31位为1）。

```c++
#define	__FD_MASK(d)	((__fd_mask) (1UL << ((d) % __NFDBITS)))
```

* \_\_FDS_BITS(set), 即表示引用set结构中的fds_bits成员， 即可以直接引用fd_set集合中的fds_bits成员。

```
# define __FDS_BITS(set) ((set)->fds_bits)
```



接下来，就是将上面的介绍的串起来。

对于**FD_SET**宏，定义如下

```c++
#define	FD_SET(fd, fdsetp)	__FD_SET (fd, fdsetp)
```

接着看, 下面的转换

```c++
#define __FD_SET(d, set) \
  ((void) (__FDS_BITS (set)[__FD_ELT (d)] |= __FD_MASK (d)))
```

等价于

```
void (set)->fds_bits[__FD_ELT (d)] |= __FD_MASK (d)))
```

意味着\_\_FD_set(d , set)， 将描述符d存放到结构set中的fds\_bits中，第(d) / \_\_NFDBITS号元素中第1UL << ((d) % __NFDBITS))位赋值位1。（详细前面有分析）



同理可分析**\_\_FD_CLR, \_\_FD_ISSET**宏。注意区分他们各自使用的**位运算**.



对于**\_\_FD_ZERO**宏，主要将描述符集合fd_set清空（即所有位置0）。 其实现有两种不同方式。 （基于我们对前面结构的了解，如果我们自己实现也不难）。第一个是GNU CC标准，目前看不太懂。第二个版本简单遍历数组，将每个元素置0.

```c++
#if defined __GNUC__ && __GNUC__ >= 2

# if __WORDSIZE == 64
#  define __FD_ZERO_STOS "stosq"
# else
#  define __FD_ZERO_STOS "stosl"
# endif

# define __FD_ZERO(fdsp) \
  do {									      \
    int __d0, __d1;							      \
    __asm__ __volatile__ ("cld; rep; " __FD_ZERO_STOS			      \
			  : "=c" (__d0), "=D" (__d1)			      \
			  : "a" (0), "0" (sizeof (fd_set)		      \
					  / sizeof (__fd_mask)),	      \
			    "1" (&__FDS_BITS (fdsp)[0])			      \
			  : "memory");					      \
  } while (0)

#else	/* ! GNU CC */

/* We don't use `memset' because this would require a prototype and
   the array isn't too big.  */
# define __FD_ZERO(set)  \
  do {									      \
    unsigned int __i;							      \
    fd_set *__arr = (set);						      \
    for (__i = 0; __i < sizeof (fd_set) / sizeof (__fd_mask); ++__i)	      \
      __FDS_BITS (__arr)[__i] = 0;					      \
  } while (0)

#endif	/* GNU CC */
```



# 三、总结

* 为什么FD_相关宏实现使用了位数组？

  我觉得主要是为了节约内存。每一个二进制位即可用来表示一个描述符， 1个字节可用来表示8个描述符。

* fd_set数据结构里面只有包含了一个数组成员， 为什么要使用一个结构来封装这个数组，而不直接使用这个数组即可？

  我觉得主要是为了方便表示，使用一个变量和使用一个数组，显然使用一个变量更加方便。而且在底层数据拷贝的时候，拷贝一个数据结构变量，明显比拷贝一个数组方便。

* 对于信号集函数，实现也类似，使用了位数组。 其声明函数如下。

```c++
SIGSETOPS(3)                                               Linux Programmer's Manual                                               SIGSETOPS(3)

NAME
       sigemptyset, sigfillset, sigaddset, sigdelset, sigismember - POSIX signal set operations

SYNOPSIS
       #include <signal.h>

       int sigemptyset(sigset_t *set);

       int sigfillset(sigset_t *set);

       int sigaddset(sigset_t *set, int signum);

       int sigdelset(sigset_t *set, int signum);
```



# 四、完整源码

头文件/usr/include/x86_64-linux-gnu/sys/select.h 

```c++
/*	POSIX 1003.1g: 6.2 Select from File Descriptor Sets <sys/select.h>  */

#ifndef _SYS_SELECT_H
#define _SYS_SELECT_H	1

#include <features.h>

/* Get definition of needed basic types.  */
#include <bits/types.h>

/* Get __FD_* definitions.  */
#include <bits/select.h>

/* Get __sigset_t.  */
#include <bits/sigset.h>

#ifndef __sigset_t_defined
# define __sigset_t_defined
typedef __sigset_t sigset_t;
#endif

/* Get definition of timer specification structures.  */
#define __need_time_t
#define __need_timespec
#include <time.h>
#define __need_timeval
#include <bits/time.h>

#ifndef __suseconds_t_defined
typedef __suseconds_t suseconds_t;
# define __suseconds_t_defined
#endif


/* The fd_set member is required to be an array of longs.  */
typedef long int __fd_mask;

/* Some versions of <linux/posix_types.h> define this macros.  */
#undef	__NFDBITS
/* It's easier to assume 8-bit bytes than to get CHAR_BIT.  */
#define __NFDBITS	(8 * (int) sizeof (__fd_mask))
#define	__FD_ELT(d)	((d) / __NFDBITS)
#define	__FD_MASK(d)	((__fd_mask) (1UL << ((d) % __NFDBITS)))

/* fd_set for select and pselect.  */
typedef struct
  {
    /* XPG4.2 requires this member name.  Otherwise avoid the name
       from the global namespace.  */
#ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->fds_bits)
#else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->__fds_bits)
#endif
  } fd_set;

/* Maximum number of file descriptors in `fd_set'.  */
#define	FD_SETSIZE		__FD_SETSIZE

#ifdef __USE_MISC
/* Sometimes the fd_set member is assumed to have this type.  */
typedef __fd_mask fd_mask;

/* Number of bits per word of `fd_set' (some code assumes this is 32).  */
# define NFDBITS		__NFDBITS
#endif


/* Access macros for `fd_set'.  */
#define	FD_SET(fd, fdsetp)	__FD_SET (fd, fdsetp)
#define	FD_CLR(fd, fdsetp)	__FD_CLR (fd, fdsetp)
#define	FD_ISSET(fd, fdsetp)	__FD_ISSET (fd, fdsetp)
#define	FD_ZERO(fdsetp)		__FD_ZERO (fdsetp)


__BEGIN_DECLS

/* Check the first NFDS descriptors each in READFDS (if not NULL) for read
   readiness, in WRITEFDS (if not NULL) for write readiness, and in EXCEPTFDS
   (if not NULL) for exceptional conditions.  If TIMEOUT is not NULL, time out
   after waiting the interval specified therein.  Returns the number of ready
   descriptors, or -1 for errors.

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int select (int __nfds, fd_set *__restrict __readfds,
		   fd_set *__restrict __writefds,
		   fd_set *__restrict __exceptfds,
		   struct timeval *__restrict __timeout);

#ifdef __USE_XOPEN2K
/* Same as above only that the TIMEOUT value is given with higher
   resolution and a sigmask which is been set temporarily.  This version
   should be used.

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int pselect (int __nfds, fd_set *__restrict __readfds,
		    fd_set *__restrict __writefds,
		    fd_set *__restrict __exceptfds,
		    const struct timespec *__restrict __timeout,
		    const __sigset_t *__restrict __sigmask);
#endif


/* Define some inlines helping to catch common problems.  */
#if __USE_FORTIFY_LEVEL > 0 && defined __GNUC__
# include <bits/select2.h>
#endif

__END_DECLS

#endif /* sys/select.h */

```

头文件/usr/include/x86_64-linux-gnu/bits/select.h

```c++
#ifndef _SYS_SELECT_H
# error "Never use <bits/select.h> directly; include <sys/select.h> instead."
#endif

#include <bits/wordsize.h>


#if defined __GNUC__ && __GNUC__ >= 2

# if __WORDSIZE == 64
#  define __FD_ZERO_STOS "stosq"
# else
#  define __FD_ZERO_STOS "stosl"
# endif

# define __FD_ZERO(fdsp) \
  do {									      \
    int __d0, __d1;							      \
    __asm__ __volatile__ ("cld; rep; " __FD_ZERO_STOS			      \
			  : "=c" (__d0), "=D" (__d1)			      \
			  : "a" (0), "0" (sizeof (fd_set)		      \
					  / sizeof (__fd_mask)),	      \
			    "1" (&__FDS_BITS (fdsp)[0])			      \
			  : "memory");					      \
  } while (0)

#else	/* ! GNU CC */

/* We don't use `memset' because this would require a prototype and
   the array isn't too big.  */
# define __FD_ZERO(set)  \
  do {									      \
    unsigned int __i;							      \
    fd_set *__arr = (set);						      \
    for (__i = 0; __i < sizeof (fd_set) / sizeof (__fd_mask); ++__i)	      \
      __FDS_BITS (__arr)[__i] = 0;					      \
  } while (0)

#endif	/* GNU CC */

#define __FD_SET(d, set) \
  ((void) (__FDS_BITS (set)[__FD_ELT (d)] |= __FD_MASK (d)))
#define __FD_CLR(d, set) \
  ((void) (__FDS_BITS (set)[__FD_ELT (d)] &= ~__FD_MASK (d)))
#define __FD_ISSET(d, set) \
  ((__FDS_BITS (set)[__FD_ELT (d)] & __FD_MASK (d)) != 0)
```

