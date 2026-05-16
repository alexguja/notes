## Basic commands

```sh
# Basic zsh commands
pwd      # print working directory
ls       # list files and directories
cd DIR_NAME # change directory to DIR_NAME
cd ..    # go up one directory
cd ~     # go to home directory
cd -     # go to previous directory
echo $VARIABLE_NAME # print value of VARIABLE_NAME
which COMMAND_NAME # show path of COMMAND_NAME
date     # show current date and time
!!       # repeat last command
history  # show command history
clear    # clear terminal screen

ctrl + r # reverse search in history
ctrl + c # cancel current command
ctrl + d # terminate shell session

# Signals
ps aux | grep PROCESS_NAME # find process by name
kill -9 PID                # force kill process with PID
kill -l                    # list all signals

# Aliases
~ stands for $HOME
-9 stands for SIGKILL

# Flags 
-  # lets you pass multiple flags together (e.g., ls -la)
-- # lets you pass long-form flags (e.g., git --help)


# Interacting with files
less             # paginate and view file content
man              # show manual for a command
cat              # concatenate and display file content
head             # display the first lines of a file
tail             # display the last lines of a file
mkdir DIR_NAME   # create a new directory
touch FILE_NAME  # create a new empty file or update file's modified time
rm FILE_NAME     # remove file
rm -r DIR_NAME   # remove directory recursively with confirmation
rm -rf DIR_NAME  # force remove directory recursively without confirmation
cp src dest      # copy file or directoryß
mv old new       # move or rename file or direc
tar              # archive files into tarballs
gzip             # compress files using gzip
```

```sh
mkdir folder1
touch file1.txt file2.txt folder1/file3.txt
tar -cf archive.tar file1.txt file2.txt folder1
ls -lsah
```