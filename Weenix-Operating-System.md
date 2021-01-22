The Weenix OS
=============

Motivation
----------

The Weenix operating system is a project for people interested in
writing parts of a Unix kernel. The operating system was originally
written in 1998 by teaching assistants for Brown University’s operating
systems course, taught by Professor Tom Doeppner, and has been
maintained by that course’s staff ever since. While the operating system
is mostly based on early versions of Unix, it incorporates many recent
developments in operating systems.

This book is intended to be a guide for students who are working on the
Weenix operating system and those interested in contributing to it. Part
1 is an introduction to Weenix, including the basics of setting up your
development environment, using the build system, and running the OS in a
virtual machine. Part 2 contains five chapters, each of which specify an
assignment. In the operating systems course taught at Brown, students
have a choice between doing either:

-   All five assignments.

-   A separate threads library assignment followed by two of the Weenix
    assignments ( and ).

Students pursuing the former option are assigned a mentor from the
course staff who will be a source of clarification or hints if the
student wishes. The latter option is provided to allow students who are
not interested in a career in operating systems development to learn the
basics of operating systems without spending the better part of their
semester hacking the Weenix kernel. Each assignment chapter begins with
an overview of the assignment, then describes the expected
implementation in more detail, and ends with tips for testing. Part 3
describes the inner workings of some key parts of the kernel not
typically explored during the assignments. This is mainly a resource for
those wishing to contribute to the Weenix project. Finally, several
appendices are included which provide more details about development
tools, online resources, and the protocols of contributing new code.

How to Read This Book
---------------------

We expect that there are at least two groups of people who will want to
read this book. If you are someone who wants to see Weenix run *right
now*, jump ahead to . Everyone else should begin by reading and then can
read the assignments, manual, and appendices in whatever order they
please. However, if you plan to do the assignments, we recommend that
you do them in order, as they build on top of one another.

History
-------

The Weenix OS, and the course that it grew up with, have a long and
illustrious history. Although the Brown operating systems course has
been taught since the 1960s, Weenix was first written in the spring of
1998, running on what was known only as the Brown Simulator 2.0. The two
pieces of software were written by Keith Adams, Michael Castelle,
Caroline Dahllof, Jason Lango, and Dave Powell. The name Weenix (“little
\*nix”) was invented by Keith Adams, and the OS was designed based on
early versions of Unix. In 2004, a competing Brown operating system
based on Windows (named HipOS) was created by Hari Khalsa, whose effort
was apparently completely vanquished. During 2007-2009, Weenix moved off
of the Brown Simulator and onto Xen virtual machines, an effort
spearheaded by David Pacheco, Joel Weinberger, Dan Kuebrich, and Dimo
Bounov. In 2010, Weenix found its current home running on Bochs, a
virtual machine including a simulated real machine. Chris Siden, Alvin
Kerber, and Shaun Verch were the major contributors for this move.
Weenix has since moved onto QEMU, an x86 processor emulator.

The features of Weenix included:

-   Intelligent multitasking

-   Virtual memory

-   Terminal emulation

-   Polymorphic file system support

-   Advanced device support (including APIC and PATA with BMIDE)

In the fall of 2018, Tom and group of students sought to port Weenix to x86_64 and generally improve the Weenix operating system.
The following spring, David Armanious, made massive improvements to Weenix including:

-   Symmetric multiprocessing

-   Kernel and Userspace preemption

-   ACPI support

-   Improved VFS and VM subsystems

-   Improved VGA support

-   Much, much, more 
 

Acknowledgements
----------------

Weenix would not have been possible without the help of many earnest,
devoted individuals. Thanks first to Professor Tom Doeppner, for his
patient guidance and leadership.

Thanks also to the many course TAs since the creation of Weenix.

-   Mike Castelle, David Powell, Caroline Dahllof, Tim Rowley

-   Thomas Crulli, Matt Eccleston, Keith Adams, Matt Ahrens, Tim Rowley

-   Matt Ahrens, Adam Leventhal, Matt Amdur, Pete Demoreuille, Ioannis
    Tsochantaridis

-   Pete Demoreuille, Peter Griess, Fareed Behmaram-Mosavat, Dan Polivy,
    Dmitriy Genzel

-   Kit Colbert, Sean Cannella, Albert Huang, Eric Schrock, Mark Ture

-   Susannah Raub, Dan Stowell, Luke Peng, Pat Sunday, David Reiss

-   Mark Johnson, Stacy Wong, Sean Smith, Dan Spinosa, Hari Khalsa

-   Lucia Ballard, Eric Tamura, Edwin Chang, Pat Culhane, Adam Fenn

-   Dan Leventhal, Aaron Myers, Adam Cath, Dave Pacheco, Joel Weinberger

-   Colin Gordon, Itay Neeman, Dan Kuebrich, Aurojit Panda, Lincoln
    Quirk, Owen Strain

-   Andres Douglas, Tim O’Donnell, Dimitar Bounov, Brandon Diamond,
    Travis Fischer, Allan Shortlidge

-   Spencer Brody, Robert Mustacchi, Chris Siden, Travis Webb

-   Chris Siden, Venkatasubramanian Jayaraman, Alvin Kerber, Matt
    Mallozzi, Ryan Zelen

-   Shaun Verch, Dan Kimmel, J. Rassi, Basil Crow, Irina Calciu,
    Nickolay Ratchev, Andrew Ayer, Marcelo Martins

-   Ethan Langevin, Aswin Karumbunathan, Jordan Place, Matt Krukowski,
    Dave Kilian, Irina Calciu

-   Eric Caruso, Jackson Owens, James Kelley, Joel Nackman, Eli Wald,
    Zhixiong Chen

-   DJ Hoffman, Ryan Roelke, Jake Ellis, Alex Kleiman, Alex Light, Indy
    Prentice, Eli Rosenthal

-   Eli Rosenthal, Ian Boros, Kyle Laracey, Sorin Vatasoiu, Jan
    Wyszynski, Xiangyu (Tom) Yao

-   Kyle Laracey, Archita Agarwal, Ian Boros, Isaac Davis, Egor
    Shakhnovskiy, Di Yang (Steven) Shi

-   Isaac Davis, Jonathan Lister, Max Luzuriaga, Benjamin Murphy, Jake
    Saferstein, Di Yang (Steven) Shi

-   Natalie Lindsay, Peter Cho, Tynan Couture-Rashid, David Armanious,
    Ethan Sattler, Gabriel Marks, Ankita Sharma, Sandy Harvie, Ben
    Navetta, Jarod Boone, and Will Povell

-   Peter Cho, Ethan Sattler, Natalie Lindsay, Amos Jackson, Justin Zhang, Wanxin Cai, David Charatan, Gabe Marks, Will Riley, Junlin Zeng

-   Ethan Sattler, Gabriel Marks, Bryan Zhang, Jason Crowley, Peter Harvie, Star Su Su, Will Hayward, Esther Kim

In particular, many thanks Keith Adams, Adam Fenn, Dave Pacheco,
Dimitar Bounov, Alvin Kerber, Chris Siden, and David Armanious for their major
contributions to the project.

Finally, thanks to the innumerable students who have implemented Weenix
over the years. We hope the knowledge you gained in the process has
served you well.

