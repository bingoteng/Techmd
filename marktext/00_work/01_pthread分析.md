# 一、api接口文档

## 1.线程创建`__pthread_create`

1. 前置描述：校验参数-分配资源-处理属性-创建线程，是所有OS线程创建接口的通用逻辑流程。本接口是在RTOS+Posix兼容+资源受限三个约束下的一个实现流程。

2. 描述：这段代码实现了一个 POSIX 兼容的线程创建接口 `_pthreadCreate`，它是对底层内核线程原语 `srh_thread_create` 的封装。

3. 功能实现主要分为8个部分：
   
   1. 参数校验：检查传入的线程指针，属性对象及其内部字段的合法性;
      
      1. 如果调用者传入属性对象&attr，要求其内部字段threadAttrStatus==PTHREAD_INITIALIZED;若内部字段threadAttrInheritsched==PTHREAD_EXPLICIT_SCHED，还要检查其优先级prio是否超过最大值。
      
      2. 若不传入属性，即就是NULL，则按照default初始化属性对象。
   
   2. 资源分配：在内核堆上分配internalPthread的结构体内存，并将其清零。
   
   3. 设置线程栈大小以及线程名称
   
   4. 调用内核启动线程，将属性传入srh_thread_attr_init
   
   5. 线程创建失败资源清理
   
   6. 线程创建成功资源收尾

4. 实现流程图：

5. 异常处理

6. 

## 2.线程退出`pthread_exit`
