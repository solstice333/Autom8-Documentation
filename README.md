#Autom8#

##Description##

Autom8 is a solution for automating sequences of shell commands 
that require automatic logging. There are three different ways of printing
to STDOUT and to the log file which affects the speed of the entire process.
Looping features exist and is based on the exit status of a subprocess. 
Internal functions in the perl script exist in the case where data needs to
be shared among the subprocesses/commands executed in the sequence. Examples
of where this application has had some usage is looping through git bisect, 
overlooking build processes, automating regression tests, and
continuous integration.


##Dependencies##

1. Windows 7/Server 2003

2. Perl 5.16.3 : http://www.activestate.com/activeperl
  - Must be installed in C:\Perl or C:\Perl64

3. Tie::IxHash 1.23 : https://code.activestate.com/ppm/Tie-IxHash/

4. File-Copy-Recursive 0.38 : https://code.activestate.com/ppm/File-Copy-Recursive/

5. Win32::Env : https://code.activestate.com/ppm/Win32-Env/

**Installation**

If on DOMAIN1, the easiest way to install Autom8 is to use 
\\\\Carson\Engineering\Users\kn\Autom8_Installer\install_Autom8.bat. This will
install the Autom8 repository with the dependencies in the path of your
choosing. For just installing dependencies, use install_dep.bat.

**Updates**

If on DOMAIN1, update.bat is used to check for Autom8 updates

##How To Use##

Usage begins with autom8.bat in order to manipulate the environment first. 
The only files that really needs to be modified by the user are templates 
which include the sequence of commands to be ran. Ordered associated arrays
(Tie::IxHash) are used to fork a subprocess shell and execute the desired 
command(s). The key represents a comment, and the value represents the 
command. Note you cannot have the same comment twice since it is, 
fundamentally, a hash table. The perl script automatically executes the 
commands listed inside the hash table within the current directory. The 
words 'Finished' and 'Exiting' are keywords reserved by autom8.bat and 
autom8_engine.pl for Inter-Process Communication purposes. If any of these keywords 
appear in console during script execution, the application will exit.


**Usage:**

```
Name
   autom8.bat 

Synopsis
   autom8 [-i] <path to template>
   autom8 -h|-c|-s
   autom8 -n <template name>
   autom8 -d <task name>

Description:
   Appends newer Perl paths to the PATH environment variable,
   and executes job_sched.pl to auto schedule any job configurations,
   then runs autom8_engine.pl to automate a sequence of shell commands 
   specified by the template. Must be ran in directory in which 
   autom8.bat exists in. Exit statuses of shell commands are caught,
   and if nonzero will either block and wait for user confirmation
   to continue, or if specified in the template, will continue to 
   the next line in the template while carrying over the previous
   exit status for potential branching.

   Options:
      -i   automatically ignore exit status prompts by selecting yes
      -c   cleans the repository by deleting logs
      -n   creates a new template called <template name>.pm
      -d   deletes <task name> from scheduled jobs
      -s   shows tasks/jobs created by Autom8
      -h   displays this help description
```


**Repository Structure**

- The "lib" folder contains perl scripts used by autom8.bat. The contents
in here should not be modified.

- The "logs" folder contains all the logs that result from post-execution. 
The date/timestamp format "log_YYMMDD_HHMMSS.LOG" is used for the logs. 
The -c option can be used to clear the "logs" folder and clean the repository
safely.

- The "resources" folder can be used to store anything needed by subprocess
commands. One example for this, may be perhaps an input file that holds data
to be redirected to STDIN.

- The "templates" folder contains all the templates that can be ran by the
user. The user can create new templates using the -n option. Essentially, 
running

  ```
  autom8 templates\<name of template>.pm
  ```

  will run that specific template.

- The "usr" folder can be used for user scripts which can be called within
template.pm. This allows for user extensibility if a perl script is desired
to be ran as part of the sequence of instructions/commands.


**Template Structure**

An example of a template can be generated using the -n option with autom8.bat.
Generally, the template structure looks like this:

```
package template;
$| = 1;

use strict;
use Cwd;
use Tie::IxHash;

<global declarations>
# Example
our $foo;

<job configuration>
# Example
tie(our %job, 'Tie::IxHash',
   'taskname' => 'foo',
   'schedule' => 'weekly'
);

sub setup {
   <This is an optional function and can be used for setup code>
}

sub body {
   <Define the %commands hash table here>
   # Example
   tie(our %commands, 'Tie::IxHash', 
      '@qcomment@i' => 'echo foo'
   );
}

sub teardown {
   <This is an optional function and can be used for teardown code>
}

1;

```

Only output from %commands get logged. 

Most user code should go in setup(), body(), or teardown(). User may
extend on global declarations/initializations and use/require statements
in global space. User may also define %job in global space. However, 
any sort of inter-process communication or reading/writing from/to 
filehandles in global space will cause undefined behavior i.e. don't use 
printf(), open(), or <> in global space.

If printing to STDOUT, make sure to end with a newline because of line
buffering.

%job must be defined in global space. If defined elsewhere, %job will be
ignored. 

%commands must be defined in body(). Defining %commands elsewhere causes
undefined behavior.


**Job Configuration**

Autom8 automatically creates and updates a Windows task based on 
%template::job. %template::job must be defined similarly to:

```
tie(our %job, 'Tie::IxHash', 
   "taskname" => "foo", 
   "schedule" => "weekly",
   "every" => "2", 
   "starttime" =>  "12:00",
   "days" =>  "mon"
);

```

where valid keys/values are as follows,

Valid 'taskname' values can be any string with no whitespace

|Valid 'schedule' values|Valid 'every' values|      
|-----------------------|--------------------|
|   'minute'            |   1 - 1439         |
|   'hourly'            |   1 - 23           |    
|   'daily'             |   1 - 365          |     
|   'weekly'            |   1 - 52           |    
|   'monthly'           |   1 - 12           |    

Valid 'starttime' values are 24 hour time 00:00 - 23:59

|Valid 'schedule' values|Valid 'days' values              |    
|-----------------------|---------------------------------|
|   'weekly'            |mon, tue, wed, thu, fri, sat, sun|      
|   'monthly'           |   1 - 31                        |   

Defining the job configuration definition and then running Autom8 will
create the job.

Deleting the job configuration definition from the template does not delete
the scheduled job. User must use the -d option to do that.

Changing the job configuration definition and then running Autom8 will update
the job.

Jobs will not run if machine is on battery power.

Jobs will automatically start minimized.

It probably goes without saying that you shouldn't have the same taskname
in two different templates that have job configurations defined. If a 
taskname is used twice in two different templates, Autom8 will associate 
the last template to be used with that taskname.

Do NOT put Autom8 in a location where it is shared by different machines.
In other words do not install Autom8 in cloud storage or on a network mapped 
drive where Autom8 is being shared by multiple machines such as Dropbox.


**Prefixing Rules inside %commands**

Inside [template name].pm in %commands initialization,

- Prefixing a comment with '@o' captures STDOUT asynchronously and tees
it to the log file and the console as the command is running. This
is slower than '@q' since the program has to dynamically change file handlers 

- Prefixing a comment with '@q' captures STDOUT and tees
it to the log file and the console. With '@q' however, user must wait 
till the command is finished before it is displayed in console or 
recorded in the log file such that data is captured one block at a time
which makes the overall runtime faster than using '@o'.

- No prefixing defaults to running the command using system() which 
doesn't capture anything. This is definitely the fastest option.

- Prefixing with '@?' enables conditional branching such that the last
command's exit status is checked. The usage is such that 

   `'<Comment>' => '<LEFT_LABEL> : <RIGHT_LABEL>'`

If the exit status is 0 (success), the script branches to the `<LEFT_LABEL>`.
Otherwise, the `<RIGHT_LABEL>` is branched to. For example,

   ```
   '@?Foo' => 'LEFT : RIGHT',        # the prior command returns 0
   <more commands here>
   'LEFT' => 'echo branched here'
   ```

Branching can be done upwards or downwards. The label specified within the
hash key, must have no whitespace i.e. the following is wrong,

   ```
   '@?Foo' => 'LEFT : RIGHT',        # the prior command returns 0
   'LEFT ' => 'echo branched here'   # error, whitespace after LEFT
   ```

The script is designed to prompt the user when the exit status of the 
previous command is non-zero. To force skip these prompts and automatically
continue, use the '-i' flag.


**Suffixing Rules inside %commands**

Inside [template name].pm in %commands initialization,

- Suffixing a comment with '@i' causes that command's exit status prompt to be
ignored if exit status is nonzero. In other words, 'yes' to continue is 
automatically chosen and the prompt doesn't block. The scope of this only
applies to the command suffixed with '@i'.

For example,

'nonzero exit@i' => 'fc'  # windows' version of linux' diff command  

will normally prompt the user if he/she wants to continue with the script due
to the nonzero exit status which by convention is a bad exit status. With the
'@i' suffix, the script will automatically continue, and thus carrying the 
exit status forward to a possible conditional statement.


**Internal functions that can be used in %commands**

Inside [template name].pm,

Usage: replace the command/value section with the function name, parentheses, and
   proper number of arguments.

```
  Example: 
     'Comment' => 'cmd_updt()',
     'Comment' => 'time_init()',
     'Comment' => 'time_final(34)',  
     'Comment' => 'sleep(120)'
     'Comment' => 'chdir(C:\Users)'
     'Comment' => 'chdir("C:\Users")'
```

Existing internal functions are:

- cmd_updt(): updates environment for the current Autom8 process

- time_init(): initializes timer
  
- time_final(int timeout): stops timer, calculates difference between start 
  and stop of timer, and updates current exit status (for conditional 
  branching use) such that 0 is returned if time difference is less than 
  or equal to |timeout|. 1 otherwise. Note there are no exit status prompts 
  for this.
  
- sleep(int seconds): blocks for |seconds| seconds
  
- chdir(`<path>`) : switches working directory to `<path>`. `<path>` may be in
  quotes or exist as a bareword.
