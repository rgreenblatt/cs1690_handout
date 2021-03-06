Tools for Working in a Large Codebase
=====================================

Version Control with `git`
--------------------------

Git is a fantastic tool for version control. There are many tutorials
available online that give a good introduction to Git, but one of the
most highly recommended is on . If you prefer to work by example, try
the tutorial.

Taking notes
------------

At risk of seeming low-tech, using code tags such as `TODO`, `XXX`, and
`FIXME` with helpful notes about what you were thinking when you noticed
something was broken or unfinished will be incredibly useful. This also
allows you to search for remaining tasks by using `grep` (or, if you’re
using Git) `git grep`.

Even simpler than that, taking notes about the code you’re developing or
drawing call trees to visualize the expected call stacks can be
incredibly helpful for learning your way around a new codebase.

Text Editors and IDEs
---------------------

Because the fights between text editors are near-religious in severity,
we won’t recommend a particular code editor here, but we do recommend
that you find a “serious” text editor with some built-in niceness to
work in while you code. These include everything from Vim and Emacs
(which typically run at the command line and can run commands without
leaving the editor) to Eclipse (which does about a thousand things in
addition to being a code editor). Some of the nice things that you’ve
probably learned to love about IDEs can be approximated quite well in
more stripped-down environments as well if you use `cscope` to index
your project.

### Integration with `cscope`

Basically every major text editor has a `cscope` extension or bundle
which allows you to do forward (“Where is this function defined?”) and
reverse (“Where is this function called?”) searches, automatic text
completion for function and variable names, etc. If you choose to use a
text editor without these functions built-in, we highly recommend using
`cscope` to make your job easier. Note that there is a `cscope` build
target which will automatically update the `cscope` index every time you
recompile (or you can also re-run it manually), so you just need to set
up your editor to use the generated index file.

Debugging with `gdb`
--------------------

If you have never used a debugger before, we highly recommend you learn
to use one, either through a graphical interface like the one in many
IDEs, or from the command line. This will be a development tool you will
not want to live without once you learn to use it.

`gdb` is the most widely used debugger for C code, with support for
kernel development using many virtual machines. There are many wonderful
reasons to use `gdb` to debug Weenix in particular, so please read the
appendix on for more information.
