# WORKtop

![rec](https://cloud.githubusercontent.com/assets/11962123/21123510/118e7232-c0d1-11e6-9a0f-65eb02fc51d2.gif)

## What is this?

WORKtop is a Unix utility that provides a rolling display of SAS WORK and UTIL directories ordered by size. It also shows the ID of the process that owns each directory and (optionally) the SAS Metadata user responsible for the process. It aims to help Administrators of the SAS® BI Platform stay on top of temporary disk usage within their environments, thereby helping prevent the instability often caused by out-of-disk conditions.

The name is a portmanteau of SAS WORK and [top](http://www.unixtop.org/), a Unix utility that provides a rolling display of top cpu using processes.

## How do I use it?

Download or clone this repository. Copy `worktop` to your server(s) and `chmod +x`. It's just a bash shell script, so feel free to look through it before you run it. 

At a minimum, you can simply copy worktop to your `saswork` directory and run it with `./worktop`. However, it works much better if you copy it to a more permanent location and then pass it some arguments:

#### Target Directory

You can point worktop to a target directory by using the -d switch. For example,to use it to monitor the directories in `/data/saswork`, run it like this:

```
worktop -d /data/saswork
```

#### Sizing (refresh) Interval

Under the covers one of the things that worktop does is run `du` at a default interval of every 30 seconds. Depending on how busy your disks get, you may want to increase this interval. You can do this by using the `-n` switch. For example, to keep an eye on `/data/saswork` and update it every five minutes (300 secs), you would run:

```
worktop -d /data/saswork -n 300
```

Use the timers provided by worktop in the top right hand corner to gauge your refresh interval. As a rule of thumb I just came up with, you could try to keep your refresh interval at least 10x the time it takes to read the disk. I guess it makes sense. 

#### Highlighting Limit

To bring them to your attention, worktop will highlight any directories that exceed a given size limit by printing them in red. In addition, it will print any directories that exceed 50% of this limit in yellow. The limit is set by using the `-g` flag, and accepts SI units for byte multiples (ie. 25M, 20G, etc.), with a default limit of 1G. For example, to keep an eye on `/data/saswork` and highlight any directories larger than 25GB, you would run:

```
worktop -d /data/saswork -g 25G
```

#### Using Sudo to Check Size

If you're on a multi-user environment, your WORKPERMS setting may prevent you from being able to read (and therefore check the size of) all of the WORK and UTIL directories that are occupying disk space. If your user is part of the `sudoers` group, worktop will let you run the disk sizing command as sudo by using the `-s` switch. For example, to keep an eye on `/data/saswork` as `sudo`, run:


```
worktop -d /data/saswork -s
```

You will be prompted for your password once for the duration of the script's operation.

#### Metadata User Reconciliation

It's highly likely that you'll be using worktop to monitor the disk usage of interactive Workspace sessions. However, if your Workspace Server is configured for Token Authentication, you may find it difficult to identify which User is responsible for the Workspace session. If so, you can do the following:

##### 1. Designate a directory for your Metadata User Index file

   If your SASWORK and SASUTIL directories are shared between multiple nodes (such as with some SAS GRID architectures), ensure that you designate a directory that is writable by all users and shares the same mount point on each node. If you have multiple nodes but they don't share WORK or UTIL, then just ensure that this directory is available with the same path on each node. In either case you could probably use a subdirectory on your SASWORK disk.

##### 2. Figure out the hostname command that matches your directory naming pattern

   Run this on one of your nodes:
   ```
    hostname -s
   ```

   Then, run this:
   ```
    hostname -f
   ```

   Then, compare the output of each of these to the hostname pattern on your work/util subdirectories, and remember which one of them matches it. You'll use this in the next step.

##### 3. Update your Workspace Server config:

Add the following snippet to your `WorkspaceServer_usermods.sh`:

```sh
METAINDEXDIR=/tmp/blah/blah
mkdir -p $METAINDEXDIR
echo $$ $(echo $METAUSER | cut -d '@' -f 1 ) >> $METAINDEXDIR/workmeta_$(hostname -s)      
```

Where `/tmp/blah/blah` is your directory from Step 1., and that last `hostname -s` or `hostname -f` command matches the one you remember from Step 2. 

*Note: This works for any other SAS Servers that use `exec` to launch SAS - ie. finish with `eval exec $cmd`, although Workspace Sessions are the big ones.*

##### 4. Tell worktop where the Metadata User Index files are

Lastly, point worktop to your `METAINDEXDIR` location from Step 1. For example, to monitor `/data/saswork` while writing indexes to `METAINDEXDIR=/tmp/blah`, run:

```
worktop -d /data/saswork -i /tmp/blah
```

This will load the Metadata User Index files for each host into an associative array. Worktop will then use it to look up the name of the Metadata user directly responsible for each temporary directory and print the relevant username alongside each directory. 

#### Logging

Optionally, worktop can write to a log file. To enable this, run:

```
worktop -d /data/saswork -l /tmp/mylog.log
```

Where `/tmp/mylog.log` is the name of the file you'd like it to write to. Worktop only logs timings at the moment, but can easily be modified to log anything really. Have a look at the code, particularly the `printlog()` function.

## What do I need in order to run worktop?

The main thing you'll need is version 4 of **bash**, which probably means RHEL6 or later. Apart from that, the script checks that the following are available:

- numfmt
- tput
- du
- sort
- cut
- xargs
- date
- man
- basename
- hostname

Most people will be able to run this script without making any changes to their system or requiring any elevated privileges (to install anything new).

### Why did you write this? Doesn't ESM already do this?

Yes, ESM regularly records the size of the same temporary directories and reconciles them to their owning processes/metadata users/batch jobs. However, as far as I'm aware it's also the only product that currently performs this rather crucial analysis. ESM is a comprehensive platform monitoring product that provides a whole host of other benefits, and instead of constantly recommending it to people as the sole current solution to what is essentially a very simple problem, I thought I'd spend a couple of days writing this utility. I figured that way I could litter the readme with constant references to our excellent commercial product, [Enterprise Session Monitor™ for SAS®](https://boemskats.com/esm) :).

Also, the kids were away for the weekend and I rarely get to code anything interesting any more.

### Contributing

This project is licensed under version 3 of the GPL licence. Feel free to add to it. 

### Support 

If you have any trouble with this script, raise a Github Ticket. 
