---
title: "Hugo Integrity Problem"
date: 2022-06-09T12:39:44+08:00
draft: false
---
## The problem of integrity
When you maintain your Hugo project both on Windows and Linux/macOS, you may get this error. 
The problem is caused by the default EOL character. 
The different EOL characters in the file will produce different SHA-256 hash codes.
So what needs to do is to force the files to have LF or CRLF at the end of the line. 

## Solution
We can use git to ensure this. Just add a .gitattributes file and type the following config in the file.
~~~
* text=auto eol=lf
~~~
Then commit this change. There may be some files already used the CRLF EOL character. We just need to run the following command.
~~~
git rm --cached -r .
git reset --hard
~~~
At last, we rebuild the site and deploy it.

## Reference
* https://yqc.im/hugo-failed-to-find-a-valid-digest-in-the-integrity-attribute/