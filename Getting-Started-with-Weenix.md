## Getting Started

To get started accept the [Weenix Assignment](https://classroom.github.com/a/qk-Rm9gN) from GitHub classroom

To run Weenix, use the following commands from the project root:
```bash
make
./weenix -n
```
(note that the `-n` is only required for the first run, more on this later)

Your Weenix should run and then panic with the following error:
```
initproc_start(): Not yet implemented: PROCS: initproc_start, file main/kmain.c, line 167
```
If this happens, you are all set to start working on [Procs|Processes-and-Threads]!

Note that although we're providing some functions, you'll still have to implement everything listed by the output of `make nyi`.

## Work from home

There are a few options if you want to work from home:
1. SSH -- Tom implemented Weenix just by SSHing from his Windows machine into the department machines, which works well if you use a terminal-based text editor. You will need to forward X windows but only when spawning the emulator for Weenix (QEMU).
2. Virtual machine -- this will be discussed further down below, in the section on "Vagrant".
3. Natively -- if you run Linux on your laptop, you may be able to install QEMU and other dependencies to build and run Weenix.

## Vagrant
If you want to work on Weenix on your own machine, we recommend using [Vagrant](http://www.vagrantup.com/). Using Vagrant and the Vagrantfile provided in the directory you cloned, you can spawn a VM running Linux with all dependencies necessary for running Weenix. You'll also need a virtualizer, such as VirtualBox or VMWare. To get this working:

1. Clone your Weenix repository from GitHub Classroom onto your computer.
2. Follow our [[Vagrant Guide]] to set up your VM.
3. Install an X server and an SSH client. Most people probably already have one installed (XQuartz on Mac; PuTTY, MobaXTerm, or Xming on Windows), particularly if you've set up SSH with X11 forwarding.
4. In your Weenix repository on your home computer, run `vagrant up` from the terminal on macOS or Linux, or the command prompt on Windows. Vagrant will traverse up until it finds the Vagrantfile at the root of the repository, create a VM with the correct specifications, and launch it in the background (headless). This will take some time.

After these steps, your virtual machine will be setup. You'll be able to SSH into the machine (with X11 forwarding), by running `vagrant ssh -- -Y` in a terminal on macOS. If you're on Windows, running this command may fail, but it will give you information that you can use to configure an SSH session in MobaXTerm or another SSH client, which you can use to SSH in to the VM. Once you're SSH'd in, run `cd /vagrant`. You should see your shared Weenix directory. You can compile and run Weenix from here.

**Note:** You'll have to change the `CC` line of `Global.mk` to equal `gcc`, and change this line back to the old path when working on department machines. Make sure to comment out the old path so you can get it back later!

When you're done working, remember to `vagrant suspend` or `vagrant halt` from the same place you ran `vagrant up`, otherwise it will continue running the VM in the background.

<!-- ## Cygwin -->
<!-- Cygwin may be used for editing the Weenix source files. However, we strongly recommend you **DO NOT** use it to transfer the Weenix source to your Windows computer. Doing so may cause issues with symbolic links. WinSCP and macOS users do not need to worry about this. If this happens, we recommend copying it from the department machines again using WinSCP. If this is impossible, you must manually copy all the files listed [here](/courses/cs167/content/cygwin_files_list.txt) from the location on the left to the one on the right. Similar symlink issues may arise if using SFTP on Linux. -->

## Cscope
Cscope is an incredibly helpful tool for writing Weenix. Cscope provides Eclipse-like code navigation for C. It can do some extremely powerful things, like jump to the definition of a function or find all references to a particular symbol in your code. Plus, it's easy to learn! You can unlock much of the power of Cscope by learning only two commands.

To learn how to use Cscope, we recommend reading [this (short) guide][1]. There are also (even shorter) guides on how to use Cscope in [Vim][2], [Emacs][3], and [Sublime Text][4].

[1]: http://cscope.sourceforge.net/large_projects.html
[2]: http://cscope.sourceforge.net/cscope_vim_tutorial.html
[3]: https://techtooltip.wordpress.com/2012/01/06/how-to-integrate-emacs-cscope-to-browse-linux-kernel-source-code/
[4]: http://tobepragmatic.blogspot.com/2013/07/know-your-tools-cscope-plugin-for.html

## Tmux
If you use a terminal-based editor like Vim or Emacs (or ed or sam), you may find it useful to have various "panes" open in the same terminal window, where there might be a couple panes with Vim open, a pane with GDB running, etc. Tmux is a "terminal multiplexer" that allows you to do just this. It is a terminal based program, and thus can be run over SSH without X forwarding. Many TAs have a lot of experience with this.

## Syntastic
Syntastic is a Vim plugin that will automatically compile your code whenever you save it, and will show you your compile errors right in Vim. Needless to say, this is extraordinarily useful.

To download and set up Syntastic, first follow the instructions in the "Installation" section of [this github page](https://github.com/scrooloose/syntastic) (the setup is mercifully short). Then, add the following line to your `~/.vimrc` file:
```
let g:syntastic_c_checkers = ['make']
```
That line will tell Syntastic to compile your code with the Makefile in the current directory rather than by using gcc directly.

## Scripts
You may have noticed that when running Weenix in GDB, the QEMU window annoyingly stays open even after you quit GDB. You've probably noticed that running Weenix requires first cd'ing into the directory containing Weenix or typing the absolute path of the Weenix executable. The TAs a couple years ago wrote two very simple functions that rectified these problems: `rw` and `wg` (short for "run Weenix" and "Weenix GDB"). The first script will run Weenix (with the `-n` argument, if provided), while the second will run Weenix with GDB (with the `-n` argument, if provided), and will close QEMU after GDB exits. Both commands can be run from any directory. Inside Vagrant, these commands are preinstalled. On other systems, copy and paste the text [here](/courses/cs167/content/weenix-scripts.txt) into your `.bashrc` to use them. You must also replace `<weenix home dir>` with the directory where you normally run `./weenix`.