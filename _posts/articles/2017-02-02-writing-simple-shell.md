---
layout: article
title: Simple Linux shell
meta: In this article you will learn how to write very simple shell in C language
category: articles
author: Tadeusz Lewandowski
author_page: https://github.com/tadeusz-lewandowski
---

This semester I've had Operating Systems lecture at my university. We've learned basics of bash and C language. It inspired me to do something more with C language and I decided to create a very simple shell that can change working directory, run simple programs from */bin* and be able to exit.
Let's code that!

## Project structure


I named the project *nekoshell*. Create */lib* directory. In */lib* create */sources* and */headers*.

* */sources* contains source files with function definitions. 
* */headers* contains header files with function declarations.

*project structure tree*

```
/nekoshell
|
-- /lib
|  |
|  -- /sources
|  |
|  -- /headers
```

We have to create 3 important files:

* main.c - command listening loop.
* nekoshell.c - all functions of shell.
* nekoshell.h - declarations of shell functions.

Scheme below describes project filesystem.

*project structure tree*

```
/nekoshell
|
-- main.c
|
-- /lib
|  |
|  -- /headers
|  |  |
|  |  -- nekoshell.h
|  |
|  -- /sources
|  |  |  
|  |  -- nekoshell.c
```

## Makefile

It may be very hard to compile this project with one long *gcc* command. *Make* program is a way more comfortable to build projects.

Create makefile in */nekoshell*

*project structure tree*

```
/nekoshell
|
-- main.c
|
-- makefile
|
-- /lib
|  |
|  -- /headers
|  |  |
|  |  -- nekoshell.h
|  |
|  -- /sources
|  |  |  
|  |  -- nekoshell.c
```

Our goal is to create two object files - *nekoshell.o* and *main.o* . After that - create one executable file *nekoshell*. We don't need to rebuild all files with every little change, so we'll seperate creation of object files. When we change *nekoshell.c* and this is the only change in project, *nekoshell.o* will be created but *make* won't recreate *main.o* because there was no change in source file.

*makefile*

``` makefile
nekoshell: main.o nekoshell.o
	gcc main.o nekoshell.o -o nekoshell

main.o: main.c
	gcc -c main.c

nekoshell.o: ./lib/sources/nekoshell.c ./lib/headers/nekoshell.h
	gcc -c ./lib/sources/nekoshell.c
```

Let's see what's going on here:

* *nekoshell* executable file compiled from *main.o* and *nekoshell.o*
* *main.o* created from *main.c*
* *nekoshell.o* created from *nekoshell.c* and *nekoshell.h*

We'll add a little improvement here - create path variables.

*makefile*

``` makefile
sources = ./lib/sources
headers = ./lib/headers

nekoshell: main.o nekoshell.o
	gcc main.o nekoshell.o -o nekoshell

main.o: main.c
	gcc -c main.c

nekoshell.o: $(sources)/nekoshell.c $(headers)/nekoshell.h
	gcc -c $(sources)/nekoshell.c
```

## Command prompt

What we always see when *bash* program starts? The command prompt! We'll start with this element. It's a nice touch.

We want to display name of the shell and current working directory path in our command prompt. It'll look like this:

``` bash
nekoshell:[/home/tadeusz/Documents/C/nekoshell]>>
```

```<unistd.h/>``` contains *getcwd* function that return current working directory path. Also we need *printf* function from ```<stdio.h>``` to print command prompt. Next we have to mix it into one function.

*nekoshell.c*

``` c
#include <stdio.h>
#include <unistd.h>

#define PATH_BUFF 256

void writePrompt(){
	char path[PATH_BUFF];
	getcwd(path, PATH_BUFF);
	
	printf("nekoshell:[%s]>> ", path);
}
```

*path* variable containing current working directory path and have 256 bytes (size is injected in preprocessing). You can adjust its size if needed.

*getcwd* puts current working directory path in *path* variable (max 256 bytes in my case (size of PATH_BUFF))

In the end we just print this with *printf* function. Notice, that we don't add *'\n'* character at the end of string because we want user command in one line with prompt.

Now we put function declaration in *nekoshell.h* and include it in *main.c*

*nekoshell.h*

``` c
void writePrompt();
```

*main.c*

``` c
#include "./lib/headers/nekoshell.h"

int main(){
	// writePrompt test
	writePrompt();
	
	return 0;
}
```

Build the project via *make* command and execute file with *./nekoshell* (all in linux terminal). If everything's ok, the command prompt will be printed.

## Command listening

Command prompt is useless if we don't parse commands. Let's code *listenCommand* function!


*nekoshell.c*

``` c
// (...)

void listenCommand(){
	writePrompt();

	char *command = NULL;
	int len = 0;
	int read = getline(&command, &len, stdin);

}
```

First, we print command prompt and after that we use *getline* function declared in ```<stdio.h>``` to aquire keyboard input.

If you set command buffer to NULL and size of *len* variable to 0, the *getline* will automatically alloc memory for the line. Function returns amount of bytes read.

Next, write a simple *while* loop for command listening in *main.c*

*main.c*

``` c
#include "./lib/headers/nekoshell.h"

int main(){

	while(1){
		listenCommand();
	}
	
	return 0;
}
```

Yeah, nekoshell is finally reading commands! Let's make this shell little more functional.

## Clearing command string from unwanted endline char

We need to remove endline character (*'\n'*) because it'll break our last argument (for example if you type *cd dir* - the last argument will be *dir\n* instead of *dir*). Function for this:

*nekoshell.c*

``` c
// (...)

void clearCommand(char *command){
	int commandLength = strlen(command);
	if(command[commandLength - 1] == '\n'){
		command[commandLength - 1] = '\0';
	}
}
```

This function takes pointer to command string in which last character need to be replaced with end of a string (\0) sign.

Create function declaration in *nekoshell.h* as well.

*nekoshell.h*

``` c
// (...)

void clearCommand(char *command);
```

## *cd* command recognition

We want to handle *cd* command. Before we create proper function we need to check if number of aquired bytes is greater than 1 (including new line character), clear this command and split into arguments. Let's code that.

*nekoshell.c*

``` c
// (...)
#include <string.h>
// (...)

void listenCommand(){
	// (...)
	if(read > 1){
			// clear command (remove newline if there is one)
			clearCommand(command);

			char *commandArguments;
			commandArguments = strtok(command, " ");
	}
}
```
We need to include ```<string.h>``` because we need *strtok* function as splitting mechanism.

*strtok* has two arguments:

* string to parse
* separator

*strtok* returns next arguments with every call (if you want to use strtok twice on the same string, pass NULL as the first argument. You'll see in the next example how to use strtok multiple times).

The *commandArguments* contains first argument of our command.

We want to check, if this string is equal to "cd". If it's true, use *strtok* again and take an argument of *cd*.

*nekoshell.c*

``` c
// (...)

void listenCommand(){
	// (...)
	if(read > 1){
		// (...)

		if(strcmp(commandArguments, "cd") == 0){

			commandArguments = strtok (NULL, " ");
			
			changeDirectory(commandArguments);
			
			return;
			
		}
	}
}
```

*strcmp* compares two strings. If the first argument is equal to "cd" (0 returned) then we take another part of command by calling *strtok* again (but with NULL as first argument). Next we call our custom *changeDirectory* function and return from listenCommand (because we are going to write other functions below *cd* handler).

## Changing directory

Changing current directory is very easy because ```<unistd.h>``` gives us *chdir* function that changes current working directory to directory specified in path argument. It's a good idea to check if *path* isn't null and handle *cd* errors. How to code this:

*nekoshell.c*

``` c
// (...)
void changeDirectory(char *path){
	if(path == NULL) return;

	int cdStatus = chdir(path);
	if(cdStatus == -1){
		perror("cd error");
	}
	
}
// (...)
```

*chdir* returns -1 if something went wrong. We handle this case printing last error with system call.

Build your program and have fun changing directory!

## Exit from shell

Second functionality - exit from shell. It is only one *if* statement with *exit* function inside.

*nekoshell.c*

``` c
// (...)
void listenCommand(){
	// (...)
	if(read > 1){
		// (...)
		if(strcmp(commandArguments, "exit") == 0){
			exit(0);
		}
	}
}
// (...)
```

The last functionality - executing programs from */bin*

The algorithm will be:

1. Create a path to */bin/```<appname>```*
2. Duplicate current process
3. In child process execute program with user arguments
4. In parent process wait for changes in child process

Start with creating path to */bin/```<appname>```*

*nekoshell.c*

``` c
// (...)
void listenCommand(){
	// (...)
	if(read > 1){
		// (...)
		char fullPath[20];
		char *appName = commandArguments;
		char *pathToProgram = "/bin/";
		strcpy(fullPath, pathToProgram);
		strcat(fullPath, commandArguments);
	}
}
// (...)
```

We create *fullPath* array for strings concatentation. We copy "/bin" string to this array and after that concat *fullPath* with *appName* (appName stores the name of app to execute). 

The result may be: */bin/sleep* (if appName is "sleep").

Next step is to duplicate process by *fork* function and execute program in child process. Let's start with creating new process and waiting for results.

*nekoshell.c*

``` c
// (...)

#include <sys/types.h>
#include <sys/wait.h>

// (...)

void listenCommand(){
	// (...)
	if(read > 1){
		// (...)
		int pid = fork();
		if(pid == 0){
			

		} else{
			int status;
			wait(&status);
		}
	}
}
// (...)
```

When *fork* is called, our app is "splitted" into two parts. In one we handle child process and in the second we handle parent process.

Both of processes will check its if statement. But the results will be different because *pid* variable is equal to 0 in child process (fork returns 0 for child process). In parent process we are waiting for changes in child process and save exit status in *status* variable.

*wait* is declared in ```<sys/types.h>``` and ```<sys/wait.h>``` so don't forget to add it on top.

Now we have to replace child process with app requested by user. I recommand use of *execvp* function because you can pass array of arguments to the program that you want to execute. Let's see how to do it:

*nekoshell.c*

``` c
// (...)

#define MAX_ARGS 20

// (...)

void listenCommand(){
	// (...)
	if(read > 1){
		// (...)
		int pid = fork();
		if(pid == 0){
			char *args[MAX_ARGS];
			args[0] = appName;
			int i;
			for(i = 1; i < MAX_ARGS - 2 && commandArguments != NULL; i++){
				commandArguments = strtok (NULL, " ");
				args[i] = commandArguments;
			}
			args[i] = 0;

			execvp(fullPath, args);
			perror("error");
			exit(1);

		} else{
			int status;
			wait(&status);
		}
	}
}
// (...)
```

The *args* is an array of pointers that will contain arguments for executed program. The first string is the name of the app. Next we load all arguments (limited by MAX_ARGS preprocessor definition).

*args[i] = 0* ends the list of arguments. It's required for proper work.

If *execvp* can't execute program then we end process with 1 exit status.

Build your shell and try to execute programs like *cat*, *ls -l* and many others!

## Shell - finished

*makefile*

``` c
sources = ./lib/sources
headers = ./lib/headers

nekoshell: main.o nekoshell.o
	gcc main.o nekoshell.o -o nekoshell

main.o: main.c
	gcc -c main.c

nekoshell.o: $(sources)/nekoshell.c $(headers)/nekoshell.h
	gcc -c $(sources)/nekoshell.c
```

*main.c*

``` c
#include "./lib/headers/nekoshell.h"

int main(){

	while(1){
		listenCommand();
	}
	
	return 0;
}
```

*nekoshell.h*

``` c
void writePrompt();

void changeDirectory(char *path);

void findCommands(char *commandString, char *arguments[]);

void clearCommand(char *command);

int listenCommand();
```

*nekoshell.c*

``` c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>

#define PATH_BUFF 256
#define MAX_ARGS 20

void writePrompt(){
	char path[PATH_BUFF];
	getcwd(path, PATH_BUFF);
	
	printf("nekoshell:[%s]>> ", path);
}

void changeDirectory(char *path){
	if(path == NULL) return;

	int cdStatus = chdir(path);
	if(cdStatus == -1){
		perror("cd error");
	}
	
}

void clearCommand(char *command){
	int commandLength = strlen(command);
	if(command[commandLength - 1] == '\n'){
		command[commandLength - 1] = '\0';
	}
}

void listenCommand(){
	writePrompt();

	char *command = NULL;
	int len = 0;
	int read = getline(&command, &len, stdin);

	if(read > 1){
		// clear command (remove new line if exist)
		clearCommand(command);

		char *commandArguments;
		commandArguments = strtok(command, " ");
		if(strcmp(commandArguments, "cd") == 0){

			commandArguments = strtok (NULL, " ");
			
			changeDirectory(commandArguments);
			
			return;
			
		}

		if(strcmp(commandArguments, "exit") == 0){
			exit(0);
		}

		// if its not shell command then its a program

		char fullPath[20];
		char *appName = commandArguments;
		char *pathToProgram = "/bin/";
		strcpy(fullPath, pathToProgram);
		strcat(fullPath, commandArguments);
		
		// run app in new process and wait for results
		int pid = fork();
		if(pid == 0){
			char *args[MAX_ARGS];
			args[0] = appName;
			int i;
			for(i = 1; i < MAX_ARGS - 2 && commandArguments != NULL; i++){
				commandArguments = strtok (NULL, " ");
				args[i] = commandArguments;
			}
			args[i] = 0;

			execvp(fullPath, args);
			perror("error");
			exit(1);

		} else{
			int status;
			wait(&status);
		}
	}

}
```

## Summary

Writing command shell was a cool experience! I encourage you to extend this code with more functions and to create something amazing!
If you have any question or problems, don't be shy to write me at tadeusz.m.lewandowski@gmail.com

Thanks for reading

Special thanks to **Thompson** for help with english grammar

Sources:

[http://bprzybylski.github.io/DSOP/cw09.html](http://bprzybylski.github.io/DSOP/cw09.html)
[http://bprzybylski.github.io/DSOP/cw10.html](http://bprzybylski.github.io/DSOP/cw10.html)
[http://bprzybylski.github.io/DSOP/cw11.html](http://bprzybylski.github.io/DSOP/cw11.html)









