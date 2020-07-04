### 线程

```C
int pthread_create(pthread_t *restrict tidp,
				   const pthread_attr_t *restrict attr,
				   void *(*start_rtn)(void *), void *restrict arg);
int pthread_join(pthread_t thread, void **rval_ptr);
					// 成功返回0，否则返回错误编号
```

