---
layout: post
comments: true
title:  "Recursively retrieving files from HDFS via Java Api"
excerpt: "You will know the differences between listStatus(), listFiles(), globStatus()."
date:   2017-04-23 02:00:00
mathjax: true
---

# Files and Dirs hierarchy in HDFS
Let's say you have this directory structure in your HDFS:

```
/parent
/parent/firstFileInParentDir.txt
/parent/secondFileInParentDir.txt

/parent/firstChildDir/
/parent/firstChildDir/firstFileInFirstChildDir.txt
/parent/firstChildDir/secondFileInFirstChildDir.txt

/parent/secondChildDir
/parent/secondChildDir/firstFileInSecondChildDir.txt
/parent/secondChildDir/secondFileInSecondChildDir.txt
```

# Retrieve all files in /parent folder and its subdirectories recursively: listFiles(...)
You need to use this function of FileSystem:
```
public RemoteIterator LocatedFileStatus  listFiles(final Path f, final boolean recursive)
```
<p class="p1"><em>recursive</em> must be set to <span style="color:#993300;">true</span>.</p>

# Get all files in this directory only: listFiles(...) or listStatus(...)
You need to use this function of FileSystem:
```
public RemoteIterator LocatedFileStatus listFiles(final Path f, final boolean recursive) throws FileNotFoundException, IOException
```
<p class="p1"><em>recursive</em> must be set to <span style="color:#993300;">false</span>.</p>

#Gets all files in the directory which has asterisk (*) in the path: globStatus(...)
You need to use this function of FileSystem:
```
public FileStatus[] globStatus(Path pathPattern) throws IOException
```
This function will convert '*' into all potential paths.

For example if pathPattern passed to this function was "parent/*/", then
```FileStatus[] fileStatuses = fs.globStatus(new Path("parent/*/"));```
Then fileStatuses will contain 2 paths in it:
```
/parent/firstChildDir/
/parent/secondChildDir/
```
After getting '*' converted into all potential paths without '*', now we can use either this function, with recursive=false for immediate files in the above directories:
```
public RemoteIterator LocatedFileStatus  listFiles(final Path f, final boolean recursive) throws FileNotFoundException, IOException
```
or this function for immediate files in the above directories:

```
public FileStatus[] listStatus(Path[] files) throws FileNotFoundException, IOException
```
Note: You need to check if the returned path is file, otherwise fs.listStatus() returns both files and directories which reside immediately in the given directory.
# Example Code

{% gist elkhand/8df3d5371c539c7fd1962153bb801c28 %}

