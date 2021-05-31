## 简单的锁实现
- 添加至xv6-user/user.h中
```C
typedef struct turnlock {
	uint flag;
} turnlock; 
```