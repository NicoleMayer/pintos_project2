# Project 2: User Programs

Group member:

11611712 杨寒梅

11611725 朱怀生

## Task1 argument passing

### 1.1 Data structure and functions

In this task, we mainly modify `process.c` and deal with strings. However, to test the correctness of our algorithm in this task, we must also implement `syscall_write` in `syscall.c`.

#### 1.1.1 Data structure

No complex data structure, only use `string`s and `int`s

#### 1.1.1 Functions

- <process.c> :
  - modify `process_execute (const char *file_name)`
  - modify `load (const char *file_name, void (**eip) (void), void **esp)`
  - modify `start_process (void *file_name_)`
  - modify `setup_stack (void **esp)`
  - add `push_argument(void **esp,int argc, int argv[])`
  - modify `process_wait (tid_t child_tid)`
- <syscall.c>
  - add `sys_write(int fd, const void *buffer, unsigned size, int* ret)`

### 1.2 Algorithms

Since the implementations of  `process_wait (tid_t child_tid)` and `syscall_write(struct intr_frame *f)` are just for testing the algorithm of argument passing. So we won't get into much details about them in this section, we'll discuss it in the task2.

<process.c> :

The function `process_execute` provides `file_name`, which includes command and arguments string. 

First, split the `file_name` to separate the command and arguments. We take this command as the new thread's name and then, pass the arguments to function `start_process`,` load` and `setup_stack`.

After analysing the original pintos code, we know that after `process_execute` creates the thread, user program doesn't execute immediately, it enters the `start_process` and this function invoke `load`, which will allocate memory for user programs. So the time to set up the stack is after `load`.

To set up the stack, we first memcpy the arguments and command name and save their address for future usage. Then add alignment, and `argv`'s address stored before also make sure `argv[argc]` is a null pointer. Next add the address of `argv`, `argc` and finally a `return` address.

### 1.3 Synchronization

When parsing the command line, we use `strtok_r` given by pintos rather than `strtok`. The difference between `strtok` and `strtok_r` is that the latter is threadsafe. The `save_ptr` in `strtok_r` is provided by the caller but  `strtok`  relies on a pointer to remember where it was looking for. Thread may be interrupted at any time, so it's important to be threadsafe.

————————begin——————————抄

During the function `laod`, we allocate page directory for the file, open the file, push the argument into the stack. According to **Task3**, the file operation syscalls do not call multiple filesystem functions concurrently. Therefore, we have to keep the file from modified or opened by other processes. We implement it by using `filesys_lock` (defined in `thread.h` which we will explain in **Task3**):

```
lock_acquire(&filesys_lock);
//loading the file
lock_release(&filesys_lock);
```

Also, according to the **Task3**, while a user process is running, the operating system must ensure that nobody can modify its executable on disk. `file_deny_write(file)` denies writes to this current-running files.

————————end——————————抄

### 1.4 Rationale

In this task, we split the command name and other arguments and pass them to the function `load` and add them into the stack with correct order. 补充？？



## Task 2 Process Control Syscalls

### 2.1 Data structures and functions

#### 2.1.1 Data structure

- <thread.h>

  - We create a new struct called `child_process`

  ```C
  struct child_process
  {
    tid_t child_id;												// thread id
    bool is_exit_called;									// whether exit is called
    bool has_been_waited;									// whether the child process has been waited
    int child_exit_status;								// pass to its parent process when it is exited
    struct list_elem elem_child_status;
  };
  ```

  - We add some new attributes to the struct `thread`

  ```C
  /* parent's thread id */
  tid_t parent_id;                    
  
  /* whether its child process successfully loaded its executable */
  bool load_success;
  
  /* monitor used to wait the child, owned by wait-syscall and the waiting for
       * child to load executable */ 抄的需要改！！！！！！！！！
  struct lock lock_child;
  struct condition cond_child;
  
  /* list of children process */
  struct list children;
  
  /* the executable of the current thread, used to deny the running executable and enable the write after thread exits. */
  struct file *exec_file;
  ```

  

#### 2.1.2 Functions

- <syscall.c>
  - add `memread_user (void *src, void *des, size_t bytes)`
  - add `get_user (const uint8_t *uaddr)`
  - modify `syscall_handler (struct intr_frame *f) `
  - add `sys_halt (void)`, `sys_exit (int)`, `sys_exec (const char *cmdline)`, `sys_write(int fd, const void *buffer, unsigned size, int* ret)`, `int sys_wait(pid_t pid);`

### 2.2 Algorithms

In this task, we need to finish 4 syscalls

```C
SYS_HALT, /* shutdown the system */
SYS_EXEC, /* start a new program with process_execute() */
SYS_WAIT, /* wait for a specific child process to exit */
SYS_PRACTICE /* adds 1 to its first argument, and returns the result */
```

<process.c>:

In function `process_execute`,  invoke `sema_down` for the current thread to synchronize its execution. If the process is executed successfully, return the child process’s tid; else return `TID_ERROR`.

In function `process_wait`, we'll check if `child_tid` exists in  `thread_current()->children_list` . If not this function will return -1, else we'll check if the child has been waited before, since the document stipulates that a child should only been waited for at most once. If it has been waited before, return -1, else set the waiting flag true. After checking, we down the semaphore `wait_sema` to block the process until the child terminates, Finally we remove the child process from `child_list` and return its `child_exit_status`.

<syscall.c>:

we have a function `syscall_handler` with switch-case to execute corresponding code. The system call type is read from the user process’s stack. However, since they are in the user process's virtual address space, we should check whether the address is pointed to a valid address before executing system calls. The specific verification includes check if the address is below `PHYS_BASE`, if the address is mapped, or if it is null. If the address is invalid, then we need to free the memory page and release any locks or semaphores from this process before exiting. 

**halt**

Implemented by function `sys_halt (void)`, just call function `shutdown_power_off`.

**exec**

Implemented by function `sys_exec (const char *cmdline)`, first to check if the file referred by `file_name` is valid. If it is invalid, return -1, else we call function `process_execute (const char *file_name)`.

**wait** 

Implemented by function `sys_wait(pid_t pid)`, first check if the argument it passed (the pid) is valid. If it is invalid, return -1, else we call function `process_wait (tid_t child_tid)` implemented in task 1.

**practice**

————————待补充————————



### 2.3 Synchronization

————————BEGIN——————————抄

The system call "exec"  will returns -1 if loading the new executable fails, so it cannot return before the new executable has completed loading.  To ensure this, we add a new atrribute ` child_load_status` to struct `thread`, child can get parent process through `parent_id` and use the function `thread_find_by_id` to get the parent process and set its status. If we save it in the child, there is no way to get the status if the hcild exits before the parent checks for it. Also, when a parent process is creating a child, the child process will down the `wait_sema` and block the parent. Once the child process finish loading, it will increase the semaphore to wake up the parent process.

Besides, the function `process_wait` also use semaphore to prevent race condition. When the parent start to wait one of its child, the `sema_down` is called and it will decrease the semaphore, and this will lead to block the parent thread and when the child process exits, the semphore will increase and the parent is waked up.

————————END——————————抄

### 2.4 Rationale

In this task, we finish three kernel system calls and one for practice. To achieve the goal, we create a new structure named `child_process` and add some atrributes to the struct `thread`. Semaphores are used in this task to prevent race condition.

## Task 3 File Operation Syscalls

### 3.1 Data structure and functions

#### 3.1.1 Data structure

<thread.h>

* We add some new attributes to the struct `thread`

```C
struct list opened_files;   修改名称！！！
int fd_count;
```

<syscall.h>

- We create a new struct called `process_file`

```C
struct process_file {
	struct file* ptr;
	int fd;
	struct list_elem elem;
};
```

#### 3.1.2 Functions

* <syscall.c>
  * modify `syscall_handler (struct intr_frame *)`

  * add syscall functions

    ```C
    int syscall_creat(struct intr_frame *f);
    int syscall_remove(struct intr_frame *f);
    int syscall_open(struct intr_frame *f);
    int syscall_filesize(struct intr_frame *f);
    int syscall_read(struct intr_frame *f);
    int syscall_write(struct intr_frame *f);
    void syscall_seek(struct intr_frame *f);
    int syscall_tell(struct intr_frame *f);
    void syscall_close(struct intr_frame *f);
    ```

  * add `check_user (const uint8_t *uaddr)`

  * add `put_user (uint8_t *udst, uint8_t byte)`

  * add `find_file_desc(struct thread *, int fd)`

### 3.2 Algorithms

In this task, we need to implement another 9 syscalls concerning file operation  syscalls: create, remove, open, filesize, read, write, seek, tell, and close. When a user program is running, we must ensure that nobody can modify its executable file on the disk. For this task, we use a global lock to ensure the file syscalls is thread safe. When every syscall is called, it must acquire lock and after then, release the lock.  

Besides, the filesystem of Pintos is not thread-safe, file operation syscalls cannot call multiple filesystem functions concurrently. To ensure this, we add a new variable `fd_count` into struct `thread` to keep the file desciptor larger than `STDIN_FILENO` and `STDOUT_FILENO`, `opened_files` is to keep a thread’s opened files.

The function `syscall_handler` will determine which kind of syscall function to execute. All arguments are already in the stack when a user program invoke a syscall. So, we just need to get the parameter from the stack. Each file syscalls will call the corresponding functions in the file system library in `filesys.c` after it acquires global file system lock, this lock will be released at last.

### 3.3 Synchronization

All the file operations are protected by the global file system lock, which can prevent I/O on the same fd at the same time. First, we'll check whether the current thread is holding the global lock `filesys_lock` . If so, we release it. Then we have to close all the file the current thread opens and free all its child. Also, we disable the interruption, when we go through `thread_current()->parent->children_list` or `thread_current()->opened_files`, to prevent unpredictable error or race condition in context switch. So they will not cause race conditions. 

### 3.4 Rationale

In this task, we finish nine kernel system calls for file systems. To achieve the goal, we create a new structure named `process_file` and add some atrributes to the struct `thread`. Locks are used in this task to prevent race condition. 还有补充吗？？？

## Results

We pass all 80 tests.

此处应有截图

## Questions

**Q1: A reflection on the project–what exactly did each member do? What went well, and what could be improved?**

- Task1 :
  - Code : Hanmei Yang
  - Report : Hanmei Yang
- Task2:
  - Code : Huaisheng Zhu & Hanmei Yang
  - Report : Hanmei Yang
- Task3:
  - Code: Huaisheng Zhu
  - Report: Huaisheng Zhu

What went well: We have a good division and cooperation. Every people focus on his/her job and if someone have problems, we discuss with each other and share our ideas. Due to the good teamwork, we finish this project quickly.

What could be improved: The time schedule for this project is really tight, we should start earier to make a more thorough design. When we finish task 1, it took many times to figure out the reason why it didn't work, that is we didn't implement the system write. Orginally we thought the thread creation is to blame and this misdirected idea affected the progress of this project.

**Q2: Does your code exhibit any major memory safety problems (especially regarding C strings), memory leaks, poor error handling, or race conditions?** 

???这个靠你写了。。过程中有木有遇到？

**Q3: Did you use consistent code style? Your code should blend in with the existing Pintos code. Check your use of indentation, your spacing, and your naming conventions.** 

Yes. Our codes are consistent with the style of existing Pintos code. Take the methods' definition for example, the return type and other keywords are in one line, and the method name and parameters are in the next line. We leave each left parentheses `(`  a space between the before character. And each left brace `{` occupies one line. In terms of naming conventions, we notice that the functions and variables in pintos both use underline-naming principle, so we also use this kind of convention.

**Q4: Is your code simple and easy to understand?** 

Yes. The codes we implement are consistent with the algorithm part discussed before, we don't add a redundant line. And as the algorithm part shows, our algorithm is simple enough and readable.

???需不需要补充？？？

**Q5: If you have very complex sections of code in your solution, did you add enough comments to explain them?** 

Yes. 

做的时候注意一下有木有很长的地方。

最后一次性加完所有注释

**Q6: Did you leave commented-out code in your final submission?** 

No. We remove them in the final version.

注意最后把多余的注释删掉！！！

**Q7: Did you copy-paste code instead of creating reusable functions?** 

No. If some codes will repeat, we encapsulate the codes into a function and just invoke the function when needed.

最好有一个例子！！

**Q8: Are your lines of source code excessively long? (more than 100 characters)** 

Maybe. 搜一搜有木有过长的？？？

**Q9: Did you re-implement linked list algorithms instead of using the provided list manipu-**
**lation**

???这个就不知道了 看最后有木有搞！！

