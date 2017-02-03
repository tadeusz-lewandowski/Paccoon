---
layout: article
title: Simple Linux shell
meta: In this article you will learn how to write very simple shell in C language
category: articles
author: Tadeusz Lewandowski
author_page: https://github.com/tadeusz-lewandowski
---

In this semester at university I had a classes of operating systems. We had some basics of bash and C language. It inspired me to do something more with C language and I decided to create a very simple shell that can change directory, run simple programs from */bin* and be able to exit.
Let's code that!

## Project structure


I named my project *nekoshell*. Create */lib* directory inside your project. In */lib* create */sources* and */headers*.

* */sources* contains source files with functions definitions. 
* */headers* contains header files with functions declarations.

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

We need to create three important files:

* main.c - runs command listening loop.
* nekoshell.c - all functions of shell.
* nekoshell.h - declarations of shell functions

Look at the scheme below where to place this files.

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

It may be very hard to compile this project with one long *gcc* command. *Make* program is comfortable way to build projects.

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

Our strategy is to create two object files - *nekoshell.o* and *main.o* and after that - create one executable file *nekoshell*. We want to separate creating of object files because we don't need to rebuild all project with every little change. When we change *nekoshell.c* and this is the only change in project, *nekoshell.o* will be created but *make* doesn't recreate *main.o* because there is no changes in source file.

*makefile*

``` makefile
nekoshell: main.o nekoshell.o
	gcc main.o nekoshell.o -o nekoshell

main.o: main.c
	gcc -c main.c

nekoshell.o: ./lib/sources/nekoshell.c ./lib/headers/nekoshell.h
	gcc -c ./lib/sources/nekoshell.c
```

See whats going on here:

* *nekoshell* executable file depends on *main.o* and *nekoshell.o*
* *main.o* depends on *main.c*
* *nekoshell.o* depends on *nekoshell.c* and *nekoshell.h*

We can add a little improvement here. Create variables for paths.

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

What we see first when *bash* program starts? The command prompt! We start coding this element first.

We want to display a name of the shell and current working directory path as our command prompt. It looks like this:

``` bash
nekoshell:[/home/tadeusz/Documents/C/nekoshell]>>
```

```<unistd.h/>``` gives as *getcwd* function that returns current working directory path. We need *printf* function from ```<stdio.h>``` as well for print command prompt. How to mix it in one function:

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

*path* variable stores current working directory path and have 256 bytes of length (size is injected by preprocessor). You can increase this size if you want.

*getcwd* puts current working directory path in *path* variable (max 256 bytes (because of preprocessor))

on the end we just print this by *printf* function. Notice, that we don't create *'\n'* character at the end of string because we want to listen commands in one line just after command prompt.

Now we must put function declaration in *nekoshell.h* and include it in *main.c*

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

Now build the project via *make* command and execute file by *./nekoshell* (all in linux terminal). If everythink is ok, the command prompt will be printed.

## Listen for commands

Command prompt is useless if we don't listen and parse commands. Let's code *listenCommand* function!


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

Firstly, we write command prompt and after that we use *getline* function declared in ```<stdio.h>``` for getting user input.

If you set command buffer to NULL and size (*len* variable) to 0 the *getline* automatically alloc memory for the line. Function returns amount of the read bytes.

Write a simple *while* loop for listening commands in *mai.c*

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

Yeach, nekoshell is reading commands! Its time to create some functionalities of the shell.

## Clearing command from unwanted endline char

We need to remove endline character (*'\n'*) because it breaks our last argument (for example if you type *cd dir* - the last argument will be *dir\n* instead of *dir*). Create function for that:

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

The function takes pointer to command and replace last character with sign of end of string.

Create function declaration in *nekoshell.h* as well

*nekoshell.h*

``` c
// (...)

void clearCommand(char *command);
```

## *cd* command recognition

We want to handle *cd* command. Before we create proper function for that we need to check if read bytes of line is greater than 1 (because we count new line character), clear this command and split this into arguments. Let's code that

*nekoshell.c*

``` c
// (...)
#include <string.h>
// (...)

void listenCommand(){
	// (...)
	if(read > 1){
			// clear command (remove new line if exist)
			clearCommand(command);

			char *commandArguments;
			commandArguments = strtok(command, " ");
	}
}
```
We need to include ```<string.h>``` because we use *strtok* function as splitting mechanism.

*strtok* needs two arguments:

* string to parse
* separator

*strtok* returns next arguments with every call (if you want to use strtok again on the same string, pass NULL as the first argument. You'll see in the next example how to use strtok again).

The *commandArguments* stores first argument of our command.

We want to check, if this string is equal "cd". If its true, use *strtok* again and take an argument for *cd*.

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

*strcmp* compares two strings and returns 0 if everything is ok. If the first argument is equal "cd" then we take another part of command by calling *strtok* again (but with NULL as first argument). Next we call our custom *changeDirectory* function and return from listenCommand (because we are going to write other functions below *cd* handling).

## Changing directory

Changing current directory is very easy because ```<unistd.h>``` gives as *chdir* that changes current working directory to directory specified in path argument. Its good idea to check if *path* isn't null and handle *cd* errors. How to code this:

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

*chdir* returns -1 if something went wrong. We handle this status and print last error encountered during system call if status is equal -1

Build your program and have fun with changing directory!

## Exit from shell and executing programs

Second functionality is exit from shell. Its only one *if* statement with *exit* function inside.

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

The last functionality is executing programs from */bin*

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

We create *fullPath* array for strings concatentation. We must copy "/bin" string to this array first and after that concat *fullPath* with *appName* (appName stores the name of app to execute). 

The result may be for example */bin/sleep* (if appName is "sleep").

Next step is to duplicate process by *fork* function and execute program in child process. Let's start with creating new process and waiting for the results.

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

When *fork* is called, our app is "splitted" on two parts. In one we handle child process and second we handle parent process.

Both of processes will check if statement. But the results will be different because *pid* variable is equal 0 in child process (fork returns 0 for child process). In parent process we are waiting for changes in child process and save exit status in *status* variable.

*wait* is declared in ```<sys/types.h>``` and ```<sys/wait.h>``` so don't forget to add it on the top.

Now we have to replace child process with app defined by user in command. I recommand to use *execvp* function because you can pass array of arguments to the program that you want to execute. See how to do it:

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

The *args* is an array of pointers that will be containing arguments for executed program. The first string is the name of app. Next we load all arguments (limited by MAX_ARGS preprocessor directive).

*args[i] = 0* ends the list of arguments. Its required for proper working.

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

## Conclusion

Writting shell was cool experience! I encourage you to extend this code with more functions and create something amazing!
If you have any question or problem, don't be shy and write to me at tadeusz.m.lewandowski@gmail.com

Thanks for reading

Knowledge source:

[http://bprzybylski.github.io/DSOP/cw09.html](http://bprzybylski.github.io/DSOP/cw09.html)
[http://bprzybylski.github.io/DSOP/cw10.html](http://bprzybylski.github.io/DSOP/cw10.html)
[http://bprzybylski.github.io/DSOP/cw11.html](http://bprzybylski.github.io/DSOP/cw11.html)




