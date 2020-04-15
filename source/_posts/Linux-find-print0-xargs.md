---
title: Linux - find print0 & xargs
tags: Linux
categories: Linux
abbrlink: e855f10
date: 2016-02-20 21:59:59
---

## Errors
```
root@elastic-data:/mnt/elasticsearch/nodes/0/indices# ls -rthl
...
drwxr-xr-x   4 elasticsearch elasticsearch 4.0K Apr 15 14:55 _nGsAa--TU-Yt8zAsPWzBw
drwxr-xr-x   5 elasticsearch elasticsearch 4.0K Apr 15 14:55 -hqCV4oLRG-ttGc8A93BVw
drwxr-xr-x   5 elasticsearch elasticsearch 4.0K Apr 15 14:55 y4ORbH-YRL65u5JPRWyM3A
...

root@elastic-data:/mnt/elasticsearch/nodes/0/indices# du -sh *
du: invalid option -- '6'
du: invalid option -- 'w'
du: invalid option -- 'U'
du: invalid option -- 'g'
du: invalid option -- 'E'
du: invalid option -- 'Q'
du: invalid option -- 'q'
du: invalid option -- 'I'
du: invalid option -- 'R'
du: invalid option -- 'f'
du: eQWyw: No such file or directory
du: invalid option -- 'F'
du: xFiduSIGlLbpGulXyqA: No such file or directory
du: invalid option -- 'q'
du: invalid option -- 'C'
du: invalid option -- 'V'
du: invalid option -- '4'
du: invalid option -- 'o'
du: invalid option -- 'R'
du: invalid option -- 'G'
du: invalid option -- '-'
du: invalid -t argument 'tGc8A93BVw'
```

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

## Examples

So for the above errors, using following command check size for the directores:
```
find ./ -type d -print0 | xargs -0 du -s | sort -nk1
```

Then we can get the result without errors:
```
22790848	./TTW3kNrZQd-BETbHymRqjQ
24080308	./ENSiFN9NQe-bv3YH71xG9g
33621480	./J50jz5dFS_m6DsiVRBAEMA
35486820	./618IZDqDTnee_WoAynubXg
1202871344	./
```

More examples:
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
