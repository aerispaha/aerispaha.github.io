---
layout: post
title: File Permissions for Django Media Uploads
tags: Tech
---

After switching over a Django app's database from the SQLite starter kit to Postgres in a production environment, something happened to my ability to read/write uploaded media files. When attempting to upload an image from within my admin panel I was met with a "Error 13 - Permission denied". This kicked off an all day session trying to understand file permissions on Linux systems - here is what I learned and what worked for me (bare in mind I'm learning these things on the fly and am certainly no expert).

## Users and Groups
From what I understand, Unixy systems are set up in a way that allows multiple "users" to be able to interact with the computer simultaneously. This architecture was apparently born from the days where computers took up entire rooms and needed to be shared among many researchers. A permissions system was, therefore, needed in order to prevent people from messing with other users' files.
![Ken-Thompson-and-Dennis-Ritchie-at-PDP11]({{site.baseurl}}/assets/img/Ken-Thompson-and-Dennis-Ritchie-at-PDP11.jpg)
As it stands now, Linux files and directories can be protected by assigning read, write, and execute permissions to specific users only. Users can also be placed in "groups" that share permissions. Users and groups with limited access will not be able to interact freely with certain files and directories. With that, it seemed that in my Django app, the [Error 13] Permission Denied was being triggered because the "user" uploading the image did not have the proper permissions to write within the target folder.

## Unix File Permissions Primer
To understand how to fix this, I first checked out how permissions were set on my target media file on my server using the list command with more details:

```bash
$ ls -la /var/www/media/ #my media folder
total 8
drwxr-xr-x 2 root root 4096 Oct 24 23:15 .
drwxr-xr-x 3 root root 4096 Oct 24 23:15 ..
```

The relevant information output from `ls -la` starts in the second row after the `total 8`. The second and third rows provide information related to the current directory and parent directory, respectively (signified by the trailing `.` and `..`). The codes within the first column define how permissions are set for the directory's "owner", "group", and "other". The first letter "d", means that the row is a directory. Similarly, "r", "w", and "x" stand for read, write and execute, respectively, on that directory. After the starting "d", the order of each trio of letters spell out the permissions for the directory's owner, group, and other. Breaking down the first code `drwxr-xr-x` into its parts we get:

- d ---> this row is a directory
- rwx ---> read, write execute permissions for the owner
- r-x ---> read and execute permissions for the owner group (the "w" slot is empty)
- r-x ---> read and execute permissions for the other group (all other users can read and execute, but not write)


Next we need to understand what users are considered the owner, owner group. This information is drawn from the third and fourth columns; in this case the owner and owner group are both root and all other users are namely "other". Putting it all together, with the `ls -la` command, we understand the following permissions of my `/var/www/media/` directory:

- the owner is root and has read, write execute permissions
- the owner group is also root, having read and execute permissions only
- all other users can read and execute

## Django on nginx: users   
Now that we know how to see what users have permissions in the `/var/www/media/` directory, we need to understand which user(s) are trying to read and write. My app was set up using a Digital Ocean "one-click" Django installation on Ubuntu 14.04 on a nginx web server. nginx is like Apache in that they are web servers, and with that it (I think, I guess?) needs to define a user with set permissions on directories and files in order to serve a website.

Checking out the nginx config file on my server led me to this tidbit:

```
$ vim /etc/nginx/nginx.conf

user www-data;
worker_processes 4;
pid /run/nginx.pid;

events {
        worker_connections 768;
        ...
```

Admittedly, I don't know what most of this file means, but the first line says user www-data which I took to mean "the nginx user is named www-data". Googling around seemed to confirm that that was a correct assumption.

## Changing owners and permissions
In order for my app to be able upload media files, I needed to change the permissions and owner settings. This sounded scary at first, but doesn't have to be. [This Stack Overflow answer](http://stackoverflow.com/questions/21797372/django-errno-13-permission-denied-var-www-media-animals-user-uploads?answertab=votes#answer-21797786) was extremely helpful.  

### The easy, bad, insecure way:
```bash
$ sudo chmod -R 777 /var/www/media/
```

This command changes the permissions recursively in `/var/www/media/` to allow read, write, and execute for all users, including other. This would have solved the problem, but is not secure.

### The better way:
```bash
$ sudo groupadd varwwwusers
$ sudo adduser www-data varwwwusers
$ sudo chgrp -R varwwwusers /var/www/media/
$ sudo chmod -R 770 /var/www/media/
```

Step-by-step, this creates a new user group called 'varwwwusers', adds the user www-data to that group, changes the user group of /var/www/media/ to 'varwwwusers' and finally changes the owner privileges to 770 which allows read, write and, execute rights to the owner (root) and owner group (varwwwusers) with no access rights to anyone else. Because www-data was added to the varwwwusers group, www-data can now read and write.

With that, I fired up my app and eagerly sent an image file upload to the media directory only to run into the same [Error 13] permission denied. Desperately, I pulled out the foolish `sudo chmod -R 777 /var/www/media/` and tried again, this time with success (of course). When I inspected the uploaded file, I found this:

```bash
$ ls -la /var/www/media/ #my media folder
total 8
drwxrwxrwx 2 root    varwwwusers 4096  Oct 24 23:15 .
drwxr-xr-x 3 root    root        4096  Oct 24 23:15 ..
-rw-r--r-- 1 django  django      77105 Oct 26 20:54 cool_upload.jpg
```

So, it seemed that my `cool_upload.jpg` file was being taken by Django magic and written to my media folder as the user "django". With that in mind, I had only a few more steps wrap things up.

First, I added the user 'django' to the 'varwwwusers' group and changed the permissions back to 770:

```bash
$ sudo adduser django varwwwusers
$ sudo chmod -R 770 /var/www/media/
```

This means that directories with the owner group varwwwusers can sharedprivileges with django.
Finally, to make new files uploaded by django hand-off group ownership to varwwwusers, I needed to do this:

```bash
$ chmod g+s /var/www/media
```

which sets the group id for all new files within my media folder.

After restarting nginx and Gunicorn, BOOM my app was ready to upload with fancy file permissions and without a gaping security hole. With that, my first Linux file permissions journey was complete.

If I missed anything or misspoke (which is very likely), please let me know.
