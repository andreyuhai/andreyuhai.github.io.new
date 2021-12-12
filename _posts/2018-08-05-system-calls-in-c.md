---
title: System calls in C
tags: [c, system-calls]
categories: [TIL, C]
---

A system call is a request for service that a program makes of the kernel. An example to that would be the `system()` function.<sup>[1][1]</sup>

`int system(const char* command);`

Returns -1 on error or the return status of the command otherwise.

But since this function would run any command given to it as a string you would probably like to use something specific like `exec()` functions.

## 1. `exec()` Functions
Using `exec()` functions you can remove the ambiguity of which program to run by telling the operating system precisely which program to run.

Basically there are two types of `exec()` functions. _List functions_ and _array functions_. `exec()` functions are in `unistd.h` library.

__NOTE__: Since `exec()` functions replace the current process. After you call the `exec()` function somewhere in your code, the code below won't be executed because of the aforementioned reason. You may want to search for the `fork()` function for that, which I am not going to talk about in this article.

#### 1.1 List Functions: `execl()`, `execlp()`, `execle()`
When using list functions you just pass the arguments for the command to be executed to your function as a **list** of strings.

Take the code below for example,
```c
execl("/bin/echo","/bin/echo","here","is","a","list",NULL);
``` 
Let's take a look at what those other letters in the function name mean.


| Letter   | Meaning       |
|:--------:| :-------------|
| l | list|
| p | path, which tells the function that the command is in the path so you don't need to specify its path|
| e | environment, you can also pass some environment variables |


```c
execlp("echo","echo","this function is on the path",NULL);
```
#### Simple example using environment variables
Here is a simple example just to demonstrate how to use environment variables. 

Below is the program with which we are going to execute another program using `execle()` with some environment variables.
```c
//caller.c compiled to caller
#include <unistd.h>
int main(){
        char* env[] = {"PATH=/some/path","VERSION=1.0", NULL};
        execle("./toBeExecuted","./toBeExecuted","some parameter",NULL,env);
}
```

Here is the program which will use the environment variables sent
```c
//toBeExecuted.c compiled to toBeExecuted
#include <stdio.h> 
#include <stdlib.h>

int main(int argc, char* argv[]) {    
    printf("Version of the program: %s\n",getenv("VERSION"));  
  return 0;  
}
```
After doing `./caller` the output will be `Version of the program: 1.0`.

#### 1.2 Array Functions: `execv()`, `execvp()`, `execve()`
While using these functions you don't pass the parameters as a list, you just pass all the parameters in an array instead. Except that the usage is the same as `execl()` functions.

| Letter| Meaning              |
|:---:  |:--                   |
|v|stands for vector     |
|p|path                  |
|e|environment variables |

```c
char* args[] = {"echo","here","are","the arguments",NULL};  
execvp(args[0],args);

//or if your program is not on the PATH
char* notOnPath[] = {"/bin/echo","here","are","the arguments",NULL};  
execv(notOnPath[0],notOnPath);
```

---


There are two things I want to point: 
 1. You should always end your parameter list or array with a NULL at the end.

 2. When calling an `exec()` function the second parameter should be the same as the program name you are calling, you can check the examples above.

I have found some questions on SO about that but I couldn't really get it. Maybe one day I will.

- <a href="https://unix.stackexchange.com/questions/187666/why-do-we-have-to-pass-the-file-name-twice-in-exec-functions" target="_blank">Why do we have to pass the file name twice in exec functions?</a>
- <a href="https://stackoverflow.com/questions/2050961/is-argv0-name-of-executable-an-accepted-standard-or-just-a-common-conventi" target="_blank">Is “argv[0] = name-of-executable” an accepted standard or just a common convention?</a>



*[SO]:  StackOverflow

[1]: https://www.gnu.org/software/libc/manual/html_node/System-Calls.html
