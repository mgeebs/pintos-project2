		     +--------------------------+
		     | PROJECT 2: USER PROGRAMS |
		     |     DESIGN DOCUMENT      |
		     +--------------------------+

__---- GROUP ----__

Matthew Moltzau  
Michael Hedrick  

ARGUMENT PASSING
----------------

__---- DATA STRUCTURES ----__

##### A1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration. Identify the purpose of each in 25 words or less.

```c
/* Literally nothing applies. lol
 * I made setup_argv to convert cmdline to argc, argv.
 * I made a PUSH, POP macro in stack.h */

/* Even though it wasn't needed immediately, pargs was
 * created to pass arguments to `start_process`. */
struct pargs {
	//...
};
```

__---- ALGORITHMS ----__

##### A2: Briefly describe how you implemented argument parsing.  How do you arrange for the elements of argv[] to be in the right order? How do you avoid overflowing the stack page?

I created a `PUSH` and `POP` macro to assist me in pushing normal data types
onto the stack including pointers. However, the first thing that needs to go
on the stack are character elements of strings, and if you push those one at
a time, they end up in reverse order. For the first bit you also can't use
the `PUSH` and `POP` macro because the macro infers the size from the type,
and it makes more sense when passing a string to `PUSH`, to just push its
address.

I have not tried to prevent overflowing the stack page, but I suppose you could
do so by 1) Finding the starting address of the page that `esp` is on (just
`PHYS_BASE` in our case) then 2) Seeing if esp exceeds `pg - PGSIZE`.

__---- RATIONALE ----__

##### A3: Why does Pintos implement strtok_r() but not strtok()?

`strtok_r` is the reentrant version of `strtok`, which means it can be resumed
after being interrupted. It is the thread-safe version. We will be dealing with
a lot of interrupts in the kernel so it very important that our methods are safe
to use at all times.

##### A4: In Pintos, the kernel separates commands into a executable name and arguments. In Unix-like systems, the shell does this separation. Identify at least two advantages of the Unix approach.

The kernel is supposed to be very fast and lightweight. Giving it responsibilities
that don't directly related to running the OS should probably be done outside the
kernel. #2?

SYSTEM CALLS
------------

__---- DATA STRUCTURES ----__

##### B1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration.  Identify the purpose of each in 25 words or less.

```c
/* GENERAL SYSTEM CALL INFRASTRUCTURE */

/* Describes a generic syscall function. Even if syscalls do not fit this form
 * exactly, they will all be called through the same function pointer. */
typedef int syscall_func(int, int, int);

/* KEYED HASH TABLES */

/* Keyed hash tables were useful for both `exec` and `wait`, and file operations.
 * The beginning of any struct that wants to be used in a keyed hashtable must
 * change their structure so that they can be cast into a hash_key.
 *
 * A "keyed hash table" is really just a normal hash table, but with inline
 * helper functions so that it is clearer to use and the process is generalized.
 * The generalization is nice since we can use it for multiple things. We used
 * to have a `mfi` struct standing for "meta file info" that was equivalent to
 * the `hash_key` structure below (except it was for lists).
 */
 
 //Defined in <keyed_hash.h>
 struct hash_key {
	 int key;
	 struct hash_elem elem;
 };
 
 struct thread {
	 tid_t tid;
	 struct hash_elem hash_elem;
	 //... other thread members
 };
 
 struct file {
	 int fd;
	 //TODO this is actually a list_elem currently.
	 struct hash_elem elem;
	 //... other file members
 };

/* EXEC AND WAIT SYSTEM CALL STUFF */

/* Holds a list of a thread's currently executing children that aren't being
 * waited on. As soon as we begin waiting on a child we remove it from the
 * list so that we don't wait on it twice. */
struct hash children_hash;

/* This is how the parent actually waits on the child to exit. */
struct semaphore dying_sema;

/* This is set in sys_exit, and it is meant to be read from `process_wait` */
int exit_status;

/* This is a semaphore that allows us to read the exit status before the
 * child process is destroyed. Although we can simply yield the thread
 * to achieve the same thing, this seems to be give us a better guarantee. */
struct semaphore status_sema;

/* FILE SYSTEM CALL STUFF */

/* Helps calculate unique identifiers for files. Belongs in thread struct. */
int next_fd;

/* A list of open files per process. Also belongs in thread struct. */
struct list open_files;
```

##### B2: Describe how file descriptors are associated with open files. Are file descriptors unique within the entire OS or just within a single process?

A file descriptor is a unique identifier for an open file. A file opened multiple
times is associated with multiple file descriptors, so it is important to note
that there is a difference between a "file" and an "open file". Two different
open files may reference the same file. Note: `file` as it is defined in file.c
is an "open file". It is easier to see the relationship when you realize a file
holds a pointer to an inode, while that inode has a variable `open_cnt` to see
how many times that particular file is open. Different "open files" also have
different file positions.

I decided to make file descriptors unique just within a single process. I thought
about doing it statically, but then we'd have to synchronize incrementing unique
file descriptors. Organizing the fds per process makes it so that processes cannot
share the same "open files", which is not necessary. I guess one exception would
be stdin and stdout, which have the same fd for all processes.

__---- ALGORITHMS ----__

##### B3: Describe your code for reading and writing user data from the kernel.

We might have to update how we do things generally by using the `put_user` and
`get_user` functions provided to us. There are two ways of doing it, but Ivo
suggests using `put_user` and `get_user`.
```c
//is_mapped_user_vaddr
```

##### B4: Suppose a system call causes a full page (4,096 bytes) of data to be copied from user space into the kernel.  What is the least and the greatest possible number of inspections of the page table (e.g. calls to pagedir_get_page()) that might result?  What about for a system call that only copies 2 bytes of data?  Is there room for improvement in these numbers, and how much?

##### B5: Briefly describe your implementation of the "wait" system call and how it interacts with process termination.

If a child process is confirmed to exist for the parent, we remove
it as a child to the current process (it will die soon anyway).
Because we do this, any subsequent call to wait with the same tid
will fail since the current process will no longer have that tid.
Quick note: At the moment this isn't quite atomic, but it is
unlikely to cause a problem.

Then, we call sema down on the current thread to actually wait
until the child is exiting, where it then calls sema up.

We need to make sure that process_wait can read the return value,
so there is another semephore used so that the child waits on the
parent to signal it being done with reading the return value.
The return value is just a member of a thread.

##### B6: Any access to user program memory at a user-specified address can fail due to a bad pointer value.  Such accesses must cause the process to be terminated.  System calls are fraught with such accesses, e.g. a "write" system call requires reading the system call number from the user stack, then each of the call's three arguments, then an arbitrary amount of user memory, and any of these can fail at any point.  This poses a design and error-handling problem: how do you best avoid obscuring the primary function of code in a morass of error-handling?  Furthermore, when an error is detected, how do you ensure that all temporarily allocated resources (locks, buffers, etc.) are freed?  In a few paragraphs, describe the strategy or strategies you adopted for managing these issues.  Give an example.

__---- SYNCHRONIZATION ----__

##### B7: The "exec" system call returns -1 if loading the new executable fails, so it cannot return before the new executable has completed loading.  How does your code ensure this?  How is the load success/failure status passed back to the thread that calls "exec"?

A semaphore is used to signal when a process is finished loading.
The start process function is void, not to mention it is a part
of a whole different thread, so the status is returned via a
boolean pointer.

##### B8: Consider parent process P with child process C.  How do you ensure proper synchronization and avoid race conditions when P calls wait(C) before C exits?  After C exits?  How do you ensure that all resources are freed in each case?  How about when P terminates without waiting, before C exits?  After C exits?  Are there any special cases?

Should probably map this all out.

__---- RATIONALE ----__

##### B9: Why did you choose to implement access to user memory from the kernel in the way that you did?

##### B10: What advantages or disadvantages can you see to your design for file descriptors?
