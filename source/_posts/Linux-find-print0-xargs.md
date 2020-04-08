---
title: Linux - find print0 & xargs
categories: Linux
abbrlink: e855f10
date: 2016-02-20 21:59:59
---

## find

```
-print

True; print the full file name on the standard output, followed by a newline.   If
you are piping the output of find into another program and there is  the  faintest
possibility  that  the  files which you are searching for might contain a newline,
then you should seriously consider using the -print0  option  instead  of  -print.
See  the UNUSUAL FILENAMES section for information about how unusual characters in
filenames are handled.
```
```
-print0

True; print the full file name on the standard output, followed by a null charac‐
ter  (instead  of the newline character that -print uses).  This allows file names
that contain newlines or other types of white space to be correctly interpreted by
programs  that  process the find output.  This option corresponds to the -0 option
of xargs.
```

## xargs

Deal with the problem of Argument list too long

```
xargs - build and execute command lines from standard input

-0

Input  items  are terminated by a null character instead of by whitespace, and the
quotes and backslash are not special (every character is taken  literally).   Dis‐
ables  the  end  of file string, which is treated like any other argument.  Useful
when input items might contain white space, quote marks, or backslashes.  The  GNU
find -print0 option produces input suitable for this mode.
```

## examples

```
➜  ~ ls -rthl *.test
-rw-r--r-- 1 root root 0 Feb 20 18:21 file1.test
-rw-r--r-- 1 root root 0 Feb 20 18:21 file2.test
-rw-r--r-- 1 root root 0 Feb 20 18:21 file 3.test
-rw-r--r-- 1 root root 0 Feb 20 18:21 fi1e 4.test

➜  ~ find . -name "*.test" -print
./fi1e 3.test
./file 4.test
./file1.test
./file2.test

➜  ~ find . -name "*.test" -print0
./fi1e 3.test./file 4.test./file1.test./file2.test

➜  ~ find . -name "*.test" -print | xargs rm
rm: cannot remove ‘./fi1e’: No such file or directory
rm: cannot remove ‘3.test’: No such file or directory
rm: cannot remove ‘./file’: No such file or directory
rm: cannot remove ‘4.test’: No such file or directory

➜  ~ find . -name "*.test" -print0 | xargs -0 rm
```
