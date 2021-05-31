## 简单的 Thread Library 实现

### 修改user.h
在xv6-user/user.h中增加函数声明
```C
int create_kThread(void (*fnc)(void *), void *arg);	 
int join_kThread(void);
void init_lock(struct ticketlock *tlock);
void acquire_lock(struct ticketlock *tlock);
void release_lock(struct ticketlock *tlock);
```

### 修改ulib.c
在xv6-user/ulib.c中增加函数实现
```C
int create_kthread(void (*fnc)(void *), void *arg) {
	char *stack = sbrk(PGSIZE);
	return clone(fnc,arg,stack);	
} 

int join_kthread(void) {
    	return join();
}

void init_lock(struct turnlock *xlock) {
	xlock->flag = 0;
}

void acquire_lock(struct turnlock *xlock) {
	while(xchg(&xlock->flag, 1) != 0);
}

void release_lock(struct turnlock *xlock) {
	xchg(&xlock->flag, 0);
}

```