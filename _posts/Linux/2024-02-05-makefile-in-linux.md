---
title: Makefile in Linux
date: 2024-02-05
categories: [Linux]
tags: [linux, development]     # TAG names should always be lowercase
---
Why I need this file?
=====================

IF I DON'T USE SOMETHING, I WILL FORGET THEM.

# Thanks to [Github](https://github.com/liheqian1993/GNC-Tutorial/tree/main) & [GNU make](https://www.gnu.org/software/make/manual/make.html)

+ [Introduction](https://github.com/liheqian1993/GNC-Tutorial/tree/main/0.Intro) (compile process, GNU_GCC commands)
+ C/C++ compile

# Details

+ Pattern rule

```makefile
# Define a pattern rule that compiles every .c file into a .o file
%.o : %.c
		$(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
```

Pattern rules contain a '%' in the target. This '%' matches any nonempty string, and the other characters match themselves. ‘%’ in a prerequisite of a pattern rule stands for the same stem that was matched by the ‘%’ in the target.

Here's another example:

```makefile
# Define a pattern rule that has no pattern in the prerequisites.
# This just creates empty .c files when needed.
%.c:
   touch $@
```

+ Different `@`

  + Add an @ before a command to stop it from being printed
    You can also run make with -s to add an @ before each line

  ```makefile
  all: 
    @echo "This make line will not be printed"
    echo "But this will"
  ```
  + When there are multiple targets for a rule, the commands will be run for each target. $@ is an automatic variable that contains the target name.

  ```makefile
  all: f1.o f2.o

  f1.o f2.o:
    echo $@
  # Equivalent to:
  # f1.o:
  #	 echo f1.o
  # f2.o:
  #	 echo f2.o
  ```
  + Automatic Variables
    There are many automatic variables, but often only a few show up:

  ```makefile
  hey: one two
    # Outputs "hey", since this is the target name
    echo $@

    # Outputs all prerequisites newer than the target
    echo $?

    # Outputs all prerequisites
    echo $^

    touch hey

  one:
    touch one

  two:
    touch two

  clean:
    rm -f hey one two
  ```
+ Assignment

  + Recursively Expanded Variable Assignment
    The first flavor of variable is a recursively expanded variable. Variables of this sort are defined by lines using ‘=’ (see Setting Variables) or by the define directive (see Defining Multi-Line Variables). The value you specify is installed verbatim; if it contains references to other variables, these references are expanded whenever this variable is substituted (in the course of expanding some other string). When this happens, it is called recursive expansion.
    For example, it will echo ‘Huh?’: ‘\$(foo)’ expands to ‘\$(bar)’ which expands to ‘\$(ugh)’ which finally expands to ‘Huh?’.

  ```makefile
  foo = $(bar)
  bar = $(ugh)
  ugh = Huh?
  all:;echo $(foo)
  ```
  + Simply Expanded Variable Assignment, using ‘:=’ or ‘::=’.
    Both forms are equivalent in GNU make; however only the ‘::=’ form is described by the POSIX standard (support for ‘::=’ is added to the POSIX standard for POSIX Issue 8).
  + Immediately Expanded Variable Assignment
    unlike simple assignment the resulting variable is recursive: it will be re-expanded again on every use. In order to avoid unexpected results, after the value is immediately expanded it will automatically be quoted: all instances of \$ in the value after expansion will be converted into \$$. This type of assignment uses the ‘:::=’ operator. For example,
    ```makefile
    var = first
    OUT :::= $(var)
    var = second
    ```
    results in the OUT variable containing the text ‘first’, while here:

  ```makefile
  var = one$$two
  OUT :::= $(var)
  var = three$$four
  ```
  + Conditional Variable Assignment ‘?=’
    it only has an effect if the variable is not yet defined. This statement:

    ```makefile
    FOO ?= bar
    ```
    equivalent to:
    ```makefile
    ifeq ($(origin FOO), undefined)
      FOO = bar
    endif
    ```
