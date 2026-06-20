# chapter9_atomicity

> Converted from chapter9_atomicity.pdf

## Page 1

Principles of Computer 
System Design 
An Introduction 
Chapter 9 
Atomicity:  All-or-Nothing and 
Before-or-After 
Jerome H. Saltzer 
M. Frans Kaashoek 
Massachusetts Institute of Technology 
Version 5.0  
Saltzer & Kaashoek Ch. 9, p. i 
June 24, 2009 12:26 am

## Page 2

Copyright © 2009 by Jerome H. Saltzer and M. Frans Kaashoek. Some Rights Reserved. 
This work is licensed under a 
Creative Commons Attribution-Non­
commercial-Share Alike 3.0 United States License. For more information on what this 
license means, visit http://creativecommons.org/licenses/by-nc-sa/3.0/us/ 
Designations used by companies to distinguish their products are often claimed as trade­
marks or registered trademarks. In all instances in which the authors are aware of a claim, 
the product names appear in initial capital or all capital letters. All trademarks that 
appear or are otherwise referred to in this work belong to their respective owners. 
Suggestions, Comments, Corrections, and Requests to waive license restrictions: 
Please send correspondence by electronic mail to: 
Saltzer@mit.edu 
and 
kaashoek@mit.edu 
Saltzer & Kaashoek Ch. 9, p. ii 
June 24, 2009 12:26 am

## Page 3

CHAPTER 
Atomicity: All-or-Nothing 
and Before-or-After 
9 
CHAPTER CONTENTS 
Overview..........................................................................................9–2  
9.1 Atomicity...................................................................................9–4  
9.1.1 All-or-Nothing Atomicity in a Database .................................... 9–5  
9.1.2 All-or-Nothing Atomicity in the Interrupt Interface .................... 9–6  
9.1.3 All-or-Nothing Atomicity in a Layered Application ...................... 9–8  
9.1.4 Some Actions With and Without the All-or-Nothing Property ..... 9–10  
9.1.5 Before-or-After Atomicity: Coordinating Concurrent Threads .... 9–13  
9.1.6 Correctness and Serialization ............................................... 9–16  
9.1.7 All-or-Nothing and Before-or-After Atomicity .......................... 9–19  
9.2 All-or-Nothing Atomicity I: Concepts.......................................9–21  
9.2.1 Achieving All-or-Nothing Atomicity: ALL_OR_NOTHING_PUT .......... 9–21  
9.2.2 Systematic Atomicity: Commit and the Golden Rule ................ 9–27  
9.2.3 Systematic All-or-Nothing Atomicity: Version Histories ............ 9–30  
9.2.4 How Version Histories are Used ............................................ 9–37  
9.3 All-or-Nothing Atomicity II: Pragmatics ..................................9–38  
9.3.1 Atomicity Logs ................................................................... 9–39  
9.3.2 Logging Protocols ............................................................... 9–42  
9.3.3 Recovery Procedures .......................................................... 9–45  
9.3.4 Other Logging Configurations: Non-Volatile Cell Storage .......... 9–47  
9.3.5 Checkpoints ...................................................................... 9–51  
9.3.6 What if the Cache is not Write-Through? (Advanced Topic) ....... 9–53  
9.4 Before-or-After Atomicity I: Concepts .....................................9–54  
9.4.1  Achieving Before-or-After Atomicity: Simple Serialization ........ 9–54  
9.4.2  The Mark-Point Discipline .................................................... 9–58  
9.4.3  Optimistic Atomicity: Read-Capture (Advanced Topic) ............. 9–63  
9.4.4  Does Anyone Actually Use Version Histories for Before-or-After   
Atomicity? ........................................................................ 9–67  
9.5 Before-or-After Atomicity II: Pragmatics ................................9–69  
9.5.1 Locks ............................................................................... 9–70  
9.5.2 Simple Locking .................................................................. 9–72  
9.5.3 Two-Phase Locking ............................................................. 9–73  9–1  
Saltzer & Kaashoek Ch. 9, p. 1  
June 24, 2009 12:26 am

## Page 4

9–2 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
9.5.4  Performance Optimizations .................................................. 9–75  
9.5.5  Deadlock; Making Progress .................................................. 9–76  
9.6 Atomicity across Layers and Multiple Sites..............................9–79  
9.6.1  Hierarchical Composition of Transactions ............................... 9–80  
9.6.2  Two-Phase Commit ............................................................. 9–84  
9.6.3  Multiple-Site Atomicity: Distributed Two-Phase Commit ........... 9–85  
9.6.4  The Dilemma of the Two Generals ........................................ 9–90  
9.7 A More Complete Model of Disk Failure (Advanced Topic) .......9–92  
9.7.1  Storage that is Both All-or-Nothing and Durable ..................... 9–92  
9.8 Case Studies: Machine Language Atomicity.............................9–95  
9.8.1  Complex Instruction Sets: The General Electric 600 Line ......... 9–95  
9.8.2  More Elaborate Instruction Sets: The IBM System/370 ............ 9–96  
9.8.3  The Apollo Desktop Computer and the Motorola M68000 
Microprocessor .................................................................. 9–97 
Exercises........................................................................................9–98 
Glossary for Chapter 9 .................................................................9–107 
Index of Chapter 9 .......................................................................9–113 
Last chapter page 9–115 
Overview 
This chapter explores two closely related system engineering design strategies. The first 
is all-or-nothing atomicity, a design strategy for masking failures that occur while inter­
preting programs. The second is before-or-after atomicity, a design strategy for 
coordinating concurrent activities. Chapter 8[on-line] introduced failure masking, but 
did not show how to mask failures of running programs. Chapter 5 introduced coordi­
nation of concurrent activities, and presented solutions to several specific problems, but 
it did not explain any systematic way to ensure that actions have the before-or-after prop­
erty. This chapter explores ways to systematically synthesize a design that provides both 
the all-or-nothing property needed for failure masking and the before-or-after property 
needed for coordination. 
Many useful applications can benefit from atomicity. For example, suppose that you 
are trying to buy a toaster from an Internet store. You click on the button that says “pur­
chase”, but before you receive a response the power fails. You would like to have some 
assurance that, despite the power failure, either the purchase went through properly or 
that nothing happen at all. You don’t want to find out later that your credit card was 
charged but the Internet store didn’t receive word that it was supposed to ship the 
toaster. In other words, you would like to see that the action initiated by the “purchase” 
button be all-or-nothing despite the possibility of failure. And if the store has only one 
toaster in stock and two customers both click on the “purchase” button for a toaster at 
about the same time, one of the customers should receive a confirmation of the purchase, 
and the other should receive a “sorry, out of stock” notice. It would be problematic if 
Saltzer & Kaashoek Ch. 9, p. 2  
June 24, 2009 12:26 am

## Page 5

Overview 9–3  
both customers received confirmations of purchase. In other words, both customers 
would like to see that the activity initiated by their own click of the “purchase” button 
occur either completely before or completely after any other, concurrent click of a “pur­
chase” button. 
The single conceptual framework of atomicity provides a powerful way of thinking 
about both all-or-nothing failure masking and before-or-after sequencing of concurrent 
activities. Atomicity is the performing of a sequence of steps, called actions, so that they 
appear to be done as a single, indivisible step, known in operating system and architec­
ture literature as an atomic action and in database management literature as a transaction. 
When a fault causes a failure in the middle of a correctly designed atomic action, it will 
appear to the invoker of the atomic action that the atomic action either completed suc­
cessfully or did nothing at all—thus an atomic action provides all-or-nothing atomicity. 
Similarly, when several atomic actions are going on concurrently, each atomic action will 
appear to take place either completely before or completely after every other atomic 
action—thus an atomic action provides before-or-after atomicity. Together, all-or-noth­
ing atomicity and before-or-after atomicity provide a particularly strong form of 
modularity: they hide the fact that the atomic action is actually composed of multiple 
steps. 
The result is a sweeping simpliﬁcation in the description of the possible states of a sys­
tem. This simplification provides the basis for a methodical approach to recovery from 
failures and coordination of concurrent activities that simplifies design, simplifies under­
standing for later maintainers, and simplifies verification of correctness. These desiderata 
are particularly important because errors caused by mistakes in coordination usually 
depend on the relative timing of external events and among different threads. When a 
timing-dependent error occurs, the difficulty of discovering and diagnosing it can be 
orders of magnitude greater than that of finding a mistake in a purely sequential activity. 
The reason is that even a small number of concurrent activities can have a very large 
number of potential real time sequences. It is usually impossible to determine which of 
those many potential sequences of steps preceded the error, so it is effectively impossible 
to reproduce the error under more carefully controlled circumstances. Since debugging 
this class of error is so hard, techniques that ensure correct coordination a priori are par­
ticularly valuable. 
The remarkable thing is that the same systematic approach—atomicity—to failure 
recovery also applies to coordination of concurrent activities. In fact, since one must be 
able to deal with failures while at the same time coordinating concurrent activities, any 
attempt to use different strategies for these two problems requires that the strategies be 
compatible. Being able to use the same strategy for both is another sweeping 
simpliﬁcation. 
Atomic actions are a fundamental building block that is widely applicable in com­
puter system design. Atomic actions are found in database management systems, in 
register management for pipelined processors, in file systems, in change-control systems 
used for program development, and in many everyday applications such as word proces­
sors and calendar managers. 
Saltzer & Kaashoek Ch. 9, p. 3 
June 24, 2009 12:26 am

## Page 6

9–4 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
Sidebar 9.1:  Actions and transactions The terminology used by system designers to discuss 
atomicity can be confusing because the concept was identiﬁed and developed independently 
by database designers and by hardware architects. 
An action that changes several data values can have any or all of at least four independent 
properties: it can be all-or-nothing (either all or none of the changes happen), it can be before-
or-after (the changes all happen either before or after every concurrent action), it can be 
constraint-maintaining (the changes maintain some speciﬁed invariant), and it can be durable 
(the changes last as long as they are needed). 
Designers of database management systems customarily are concerned only with actions that 
are both all-or-nothing and before-or-after, and they describe such actions as transactions. In 
addition, they use the term atomic primarily in reference to all-or-nothing atomicity. On the 
other hand, hardware processor architects customarily use the term atomic to describe an action 
that exhibits before-or-after atomicity. 
This book does not attempt to change these common usages. Instead, it uses the qualiﬁed terms 
“all-or-nothing atomicity” and “before-or-after atomicity.” The unqualiﬁed term “atomic” may 
imply all-or-nothing, or before-or-after, or both, depending on the context. The text uses the 
term “transaction” to mean an action that is both all-or-nothing and before-or-after. 
All-or-nothing atomicity and before-or-after atomicity are universally deﬁned properties of 
actions, while constraints are properties that different applications deﬁne in different ways. 
Durability lies somewhere in between because different applications have different durability 
requirements. At the same time, implementations of constraints and durability usually have a 
prerequisite of atomicity. Since the atomicity properties are modularly separable from the other 
two, this chapter focuses just on atomicity. Chapter 10[on-line] then explores how a designer 
can use transactions to implement constraints and enhance durability. 
The sections of this chapter define atomicity, examine some examples of atomic 
actions, and explore systematic ways of achieving atomicity: version histories, logging, and 
locking protocols. Chapter 10[on-line] then explores some applications of atomicity. Case 
studies at the end of both chapters provide real-world examples of atomicity as a tool for 
creating useful systems. 
9.1 Atomicity 
Atomicity is a property required in several different areas of computer system design. 
These areas include managing a database, developing a hardware architecture, specifying 
the interface to an operating system, and more generally in software engineering. The 
table below suggests some of the kinds of problems to which atomicity is applicable. In 
Saltzer & Kaashoek Ch. 9, p. 4 
June 24, 2009 12:26 am

## Page 7

9.1 Atomicity 9–5 
this chapter we will encounter examples of both kinds of atomicity in each of these dif­
ferent areas. 
Area 
All-or-nothing atomicity 
Before-or-after atomicity 
database management 
updating more than one record 
records shared between threads 
hardware architecture 
handling interrupts and exceptions 
register renaming 
operating systems 
supervisor call interface 
printer queue 
software engineering 
handling faults in layers 
bounded buffer 
9.1.1 All-or-Nothing Atomicity in a Database 
As a first example, consider a database of bank accounts. We define a procedure named 
TRANSFER that debits one account and credits a second account, both of which are stored 
on disk, as follows: 
1 
procedure TRANSFER (debit_account, credit_account, amount) 
2 
GET (dbdata, debit_account) 
3 
dbdata ← dbdata - amount 
4 
PUT (dbdata, debit_account) 
5 
GET (crdata, credit_account) 
6 
crdata ← crdata + amount 
7 
PUT (crdata, credit_account) 
where debit_account and credit_account identify the records for the accounts to be deb­
ited and credited, respectively. 
Suppose that the system crashes while executing the PUT instruction on line 4. Even if 
we use the MORE_DURABLE_PUT described in Section 8.5.4, a system crash at just the wrong 
time may cause the data written to the disk to be scrambled, and the value of 
debit_account lost. We would prefer that either the data be completely written to the disk 
or nothing be written at all. That is, we want the PUT instruction to have the all-or-noth­
ing atomicity property. Section 9.2.1 will describe a way to do that. 
There is a further all-or-nothing atomicity requirement in the TRANSFER procedure. 
Suppose that the PUT on line 4 is successful but that while executing line 5 or line 6 the 
power fails, stopping the computer in its tracks. When power is restored, the computer 
restarts, but volatile memory, including the state of the thread that was running the 
TRANSFER procedure, has been lost. If someone now inquires about the balances in 
debit_account and in credit_account things will not add up properly because 
debit_account has a new value but credit_account has an old value. One might suggest 
postponing the first PUT to be just before the second one, but that just reduces the win­
dow of vulnerability, it does not eliminate it—the power could still fail in between the 
two PUTs. To eliminate the window, we must somehow arrange that the two PUT instruc­
tions, or perhaps even the entire TRANSFER procedure, be done as an all-or-nothing atomic 
Saltzer & Kaashoek Ch. 9, p. 5 
June 24, 2009 12:26 am

## Page 8

9–6 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
action. In Section 9.2.3 we will devise a TRANSFER procedure that has the all-or-nothing 
property, and in Section 9.3 we will see some additional ways of providing the property. 
9.1.2 All-or-Nothing Atomicity in the Interrupt Interface 
A second application for all-or-nothing atomicity is in the processor instruction set inter­
face as seen by a thread. Recall from Chapters 2 and 5 that a thread normally performs 
actions one after another, as directed by the instructions of the current program, but that 
certain events may catch the attention of the thread’s interpreter, causing the interpreter, 
rather than the program, to supply the next instruction. When such an event happens, a 
different program, running in an interrupt thread, takes control. 
If the event is a signal arriving from outside the interpreter, the interrupt thread may 
simply invoke a thread management primitive such as ADVANCE, as described in Section 
5.6.4, to alert some other thread about the event. For example, an I/O operation that the 
other thread was waiting for may now have completed. The interrupt handler then 
returns control to the interrupted thread. This example requires before-or-after atomicity 
between the interrupt thread and the interrupted thread. If the interrupted thread was in 
the midst of a call to the thread manager, the invocation of ADVANCE by the interrupt 
thread should occur either before or after that call. 
Another possibility is that the interpreter has detected that something is going wrong 
in the interrupted thread. In that case, the interrupt event invokes an exception handler, 
which runs in the environment of the original thread. (Sidebar 9.2 offers some exam­
ples.) The exception handler either adjusts the environment to eliminate some problem 
(such as a missing page) so that the original thread can continue, or it declares that the 
original thread has failed and terminates it. In either case, the exception handler will need 
to examine the state of the action that the original thread was performing at the instant 
of the interruption—was that action finished, or is it in a partially done state? 
Ideally, the handler would like to see an all-or-nothing report of the state: either the 
instruction that caused the exception completed or it didn’t do anything. An all-or-noth­
ing report means that the state of the original thread is described entirely with values 
belonging to the layer in which the exception handler runs. An example of such a value 
is the program counter, which identifies the next instruction that the thread is to execute. 
An in-the-middle report would mean that the state description involves values of a lower 
layer, probably the operating system or the hardware processor itself. In that case, know­
ing the next instruction is only part of the story; the handler would also need to know 
which parts of the current instruction were executed and which were not. An example 
might be an instruction that increments an address register, retrieves the data at that new 
address, and adds that data value to the value in another register. If retrieving the data 
causes a missing-page exception, the description of the current state is that the address 
register has been incremented but the retrieval and addition have not yet been per­
formed. Such an in-the-middle report is problematic because after the handler retrieves 
the missing page it cannot simply tell the processor to jump to the instruction that 
failed—that would increment the address register again, which is not what the program-
Saltzer & Kaashoek Ch. 9, p. 6 
June 24, 2009 12:26 am

## Page 9

9.1 Atomicity 9–7  
Sidebar 9.2:  Events that might lead to invoking an exception handler 
1.  A hardware fault occurs: 
•   The processor detects a memory parity fault. 
•   A sensor reports that the electric power has failed; the energy left in the power supply 
may be just enough to perform a graceful shutdown. 
2.  A hardware or software interpreter encounters something in the program that is clearly 
wrong: 
•   The program tried to divide by zero. 
•   The program supplied a negative argument to a square root function. 
3.  Continuing requires some resource allocation or deferred initialization: 
•   The running thread encountered a missing-page exception in a virtual memory system. 
•  The running thread encountered an indirection exception, indicating that it 
encountered an unresolved procedure linkage in the current program. 
4.  More urgent work needs to take priority, so the user wishes to terminate the thread: 
•   This program is running much longer than expected. 
•  The program is running normally, but the user suddenly realizes that it is time to 
catch the last train home. 
5. The user realizes that something is wrong and decides to terminate the thread: 
•   Calculating e, the program starts to display 3.1415… 
•   The user asked the program to copy the wrong set of files. 
6.  Deadlock: 
•   Thread A has acquired the scanner, and is waiting for memory to become free; thread 
B has acquired all available memory, and is waiting for the scanner to be released. 
Either the system notices that this set of waits cannot be resolved or, more likely, a 
timer that should never expire eventually expires. The system or the timer signals an 
exception to one or both of the deadlocked threads. 
mer expected. Jumping to the next instruction isn’t right, either, because that would 
omit the addition step. An all-or-nothing report is preferable because it avoids the need 
for the handler to peer into the details of the next lower layer. Modern processor design­
ers are generally careful to avoid designing instructions that don’t have the all-or-nothing 
property. As will be seen shortly, designers of higher-layer interpreters must be similarly 
careful. 
Sections 9.1.3 and 9.1.4 explore the case in which the exception terminates the run­
ning thread, thus creating a fault. Section 9.1.5 examines the case in which the 
interrupted thread continues, oblivious (one hopes) to the interruption. 
Saltzer & Kaashoek Ch. 9, p. 7  
June 24, 2009 12:26 am

## Page 10

9–8 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
9.1.3 All-or-Nothing Atomicity in a Layered Application 
A third example of all-or-nothing atomicity lies in the challenge presented by a fault in 
a running program: at the instant of the fault, the program is typically in the middle of 
doing something, and it is usually not acceptable to leave things half-done. Our goal is 
to obtain a more graceful response, and the method will be to require that some sequence 
of actions behave as an atomic action with the all-or-nothing property. Atomic actions 
are closely related to the modularity that arises when things are organized in layers. Lay­
ered components have the feature that a higher layer can completely hide the existence 
of a lower layer. This hiding feature makes layers exceptionally effective at error contain­
ment and for systematically responding to faults. 
To see why, recall the layered structure of the calendar management program of 
Chapter 2, reproduced in Figure 9.19.1 (that figure may seem familiar—it is a copy of 
Figure 2.10). The calendar program implements each request of the user by executing a 
sequence of Java language statements. Ideally, the user will never notice any evidence of 
the composite nature of the actions implemented by the calendar manager. Similarly, 
each statement of the Java language is implemented by several actions at the hardware 
layer. Again, if the Java interpreter is carefully implemented, the composite nature of the 
implementation in terms of machine language will be completely hidden from the Java 
programmer. 
Human user 
generating 
requests 
Calendar 
Program 
Java 
Interpreter 
Interface 
Calendar manager 
layer interface 
Java language 
layer interface 
Machine language 
layer interface 
hardware 
Typical instruction
across this interface 
Add new event on 
February 27 
nextch = instring[j]; 
add R1,R2 
FIGURE 9.1 
An application system with three layers of interpretation.The user has requested an action that 
will fail, but the failure will be discovered at the lowest layer. A graceful response involves ato­
micity at each interface. 
Saltzer & Kaashoek Ch. 9, p. 8 
June 24, 2009 12:26 am

## Page 11

9.1 Atomicity 9–9  
Now consider what happens if the hardware processor detects a condition that should 
be handled as an exception—for example, a register overflow. The machine is in the mid­
dle of interpreting an action at the machine language layer interface—an ADD instruction 
somewhere in the middle of the Java interpreter program. That ADD instruction is itself 
in the middle of interpreting an action at the Java language interface—a Java expression 
to scan an array. That Java expression in turn is in the middle of interpreting an action 
at the user interface—a request from the user to add a new event to the calendar. The 
report “Overflow exception caused by the ADD instruction at location 41574” is not intel­
ligible to the user at the user interface; that description is meaningful only at the machine 
language interface. Unfortunately, the implication of being “in the middle” of higher-
layer actions is that the only accurate description of the current state of affairs is in terms 
of the progress of the machine language program. 
The actual state of affairs in our example as understood by an all-seeing observer 
might be the following: the register overflow was caused by adding one to a register that 
contained a two’s complement negative one at the machine language layer. That 
machine language add instruction was part of an action to scan an array of characters at 
the Java layer and a zero means that the scan has reached the end of the array. The array 
scan was embarked upon by the Java layer in response to the user’s request to add an 
event on February 31. The highest-level interpretation of the overflow exception is “You 
tried to add an event on a non-existent date”. We want to make sure that this report goes 
to the end user, rather than the one about register overflow. In addition, we want to be 
able to assure the user that this mistake has not caused an empty event to be added some­
where else in the calendar or otherwise led to any other changes to the calendar. Since 
the system couldn’t do the requested change it should do nothing but report the error. 
Either a low-level error report or muddled data would reveal to the user that the action 
was composite. 
With the insight that in a layered application, we want a fault detected by a lower 
layer to be contained in a particular way we can now propose a more formal definition 
of all-or-nothing atomicity: 
All-or-nothing atomicity 
A sequence of steps is an all-or-nothing action if, from the point of view of its 
invoker, the sequence always either 
• completes, 
or 
•   aborts in such a way that it appears that the sequence had never been 
undertaken in the first place. That is, it backs out. 
In a layered application, the idea is to design each of the actions of each layer to be 
all-or-nothing. That is, whenever an action of a layer is carried out by a sequence of 
Saltzer & Kaashoek Ch. 9, p. 9  
June 24, 2009 12:26 am

## Page 12

9–10 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
actions of the next lower layer, the action either completes what it was asked to do or else 
it backs out, acting as though it had not been invoked at all. When control returns to a 
higher layer after a lower layer detects a fault, the problem of being “in the middle” of an 
action thus disappears. 
In our calendar management example, we might expect that the machine language 
layer would complete the add instruction but signal an overflow exception; the Java 
interpreter layer would, upon receiving the overflow exception might then decide that its 
array scan has ended, and return a report of “scan complete, value not found” to the cal­
endar management layer; the calendar manager would take this not-found report as an 
indication that it should back up, completely undo any tentative changes, and tell the 
user that the request to add an event on that date could not be accomplished because the 
date does not exist. 
Thus some layers run to completion, while others back out and act as though they 
had never been invoked, but either way the actions are all-or-nothing. In this example, 
the failure would probably propagate all the way back to the human user to decide what 
to do next. A different failure (e.g. “there is no room in the calendar for another event”) 
might be intercepted by some intermediate layer that knows of a way to mask it (e.g., by 
allocating more storage space). In that case, the all-or-nothing requirement is that the 
layer that masks the failure find that the layer below has either never started what was to 
be the current action or else it has completed the current action but has not yet under­
taken the next one. 
All-or-nothing atomicity is not usually achieved casually, but rather by careful design 
and specification. Designers often get it wrong. An unintelligible error message is the 
typical symptom that a designer got it wrong. To gain some insight into what is involved, 
let us examine some examples. 
9.1.4 Some Actions With and Without the All-or-Nothing Property 
Actions that lack the all-or-nothing property have frequently been discovered upon add­
ing multilevel memory management to a computer architecture, especially to a processor 
that is highly pipelined. In this case, the interface that needs to be all-or-nothing lies 
between the processor and the operating system. Unless the original machine architect 
designed the instruction set with missing-page exceptions in mind, there may be cases in 
which a missing-page exception can occur “in the middle” of an instruction, after the 
processor has overwritten some register or after later instructions have entered the pipe­
line. When such a situation arises, the later designer who is trying to add the multilevel 
memory feature is trapped. The instruction cannot run to the end because one of the 
operands it needs is not in real memory. While the missing page is being retrieved from 
secondary storage, the designer would like to allow the operating system to use the pro­
cessor for something else (perhaps even to run the program that fetches the missing 
page), but reusing the processor requires saving the state of the currently executing pro­
gram, so that it can be restarted later when the missing page is available. The problem is 
how to save the next-instruction pointer. 
Saltzer & Kaashoek Ch. 9, p. 10 
June 24, 2009 12:26 am

## Page 13

9.1 Atomicity 9–11  
If every instruction is an all-or-nothing action, the operating system can simply save 
as the value of the next-instruction pointer the address of the instruction that encoun­
tered the missing page. The resulting saved state description shows that the program is 
between two instructions, one of which has been completely executed, and the next one 
of which has not yet begun. Later, when the page is available, the operating system can 
restart the program by reloading all of the registers and setting the program counter to 
the place indicated by the next-instruction pointer. The processor will continue, starting 
with the instruction that previously encountered the missing page exception; this time it 
should succeed. On the other hand, if even one instruction of the instruction set lacks 
the all-or-nothing property, when an interrupt happens to occur during the execution of 
that instruction it is not at all obvious how the operating system can save the processor 
state for a future restart. Designers have come up with several techniques to retrofit the 
all-or-nothing property at the machine language interface. Section 9.8 describes some 
examples of machine architectures that had this problem and the techniques that were 
used to add virtual memory to them. 
A second example is the supervisor call (SVC). Section 5.3.4 pointed out that the 
SVC instruction, which changes both the program counter and the processor mode bit 
(and in systems with virtual memory, other registers such as the page map address regis­
ter), needs to be all-or-nothing, to ensure that all (or none) of the intended registers 
change. Beyond that, the SVC invokes some complete kernel procedure. The designer 
would like to arrange that the entire call, (the combination of the SVC instruction and 
the operation of the kernel procedure itself) be an all-or-thing action. An all-or-nothing 
design allows the application programmer to view the kernel procedure as if it is an exten­
sion of the hardware. That goal is easier said than done, since the kernel procedure may 
detect some condition that prevents it from carrying out the intended action. Careful 
design of the kernel procedure is thus required. 
Consider an SVC to a kernel READ procedure that delivers the next typed keystroke to 
the caller. The user may not have typed anything yet when the application program calls 
READ, so the the designer of READ must arrange to wait for the user to type something. By 
itself, this situation is not especially problematic, but it becomes more so when there is 
also a user-provided exception handler. Suppose, for example, a thread timer can expire 
during the call to READ and the user-provided exception handler is to decide whether or 
not the thread should continue to run a while longer. The scenario, then, is the user pro­
gram calls READ, it is necessary to wait, and while waiting, the timer expires and control 
passes to the exception handler. Different systems choose one of three possibilities for the 
design of the READ procedure, the last one of which is not an all-or-nothing design: 
1.  An all-or-nothing design that implements the “nothing” option (blocking read): Seeing 
no available input, the kernel procedure ﬁrst adjusts return pointers (“push the PC 
back”) to make it appear that the application program called AWAIT just ahead of its 
call to the kernel READ procedure and then it transfers control to the kernel AWAIT 
entry point. When the user ﬁnally types something, causing AWAIT to return, the 
user’s thread re-executes the original kernel call to READ, this time ﬁnding the typed 
Saltzer & Kaashoek Ch. 9, p. 11  
June 24, 2009 12:26 am

## Page 14

9–12 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
input. With this design, if a timer exception occurs while waiting, when the 
exception handler investigates the current state of the thread it ﬁnds the answer 
“the application program is between instructions; its next instruction is a call to 
READ.” This description is intelligible to a user-provided exception handler, and it 
allows that handler several options. One option is to continue the thread, meaning 
go ahead and execute the call to READ. If there is still no input, READ will again push 
the PC back and transfer control to AWAIT. Another option is for the handler to save 
this state description with a plan of restoring a future thread to this state at some 
later time. 
2.  An all-or-nothing design that implements the “all” option (non-blocking read): Seeing 
no available input, the kernel immediately returns to the application program with 
a zero-length result, expecting that the program will look for and properly handle 
this case. The program would probably test the length of the result and if zero, call 
AWAIT itself or it might ﬁnd something else to do instead. As with the previous 
design, this design ensures that at all times the user-provided timer exception 
handler will see a simple description of the current state of the thread—it is 
between two user program instructions. However, some care is needed to avoid a 
race between the call to AWAIT and the arrival of the next typed character. 
3.  A blocking read design that is neither “all” nor “nothing” and therefore not atomic: The 
kernel READ procedure itself calls AWAIT, blocking the thread until the user types a 
character. Although this design seems conceptually simple, the description of the 
state of the thread from the point of view of the timer exception handler is not 
simple. Rather than “between two user instructions”, it is “waiting for something 
to happen in the middle of a user call to kernel procedure READ”. The option of 
saving this state description for future use has been foreclosed. To start another 
thread with this state description, the exception handler would need to be able to 
request “start this thread just after the call to AWAIT in the middle of the kernel READ 
entry.” But allowing that kind of request would compromise the modularity of the 
user-kernel interface. The user-provided exception handler could equally well 
make a request to restart the thread anywhere in the kernel, thus bypassing its gates 
and compromising its security. 
The first and second designs correspond directly to the two options in the definition 
of an all-or-nothing action, and indeed some operating systems offer both options. In the 
first design the kernel program acts in a way that appears that the call had never taken 
place, while in the second design the kernel program runs to completion every time it is 
called. Both designs make the kernel procedure an all-or-nothing action, and both lead 
to a user-intelligible state description—the program is between two of its instructions— 
if an exception should happen while waiting. 
One of the appeals of the client/server model introduced in Chapter 4 is that it tends 
to force the all-or-nothing property out onto the design table. Because servers can fail 
independently of clients, it is necessary for the client to think through a plan for recovery 
Saltzer & Kaashoek Ch. 9, p. 12  
June 24, 2009 12:26 am

## Page 15

9.1 Atomicity 9–13  
from server failure, and a natural model to use is to make every action offered by a server 
all-or-nothing. 
9.1.5 Before-or-After Atomicity: Coordinating Concurrent Threads 
In Chapter 5 we learned how to express opportunities for concurrency by creating 
threads, the goal of concurrency being to improve performance by running several things 
at the same time. Moreover, Section 9.1.2 above pointed out that interrupts can also cre­
ate concurrency. Concurrent threads do not represent any special problem until their 
paths cross. The way that paths cross can always be described in terms of shared, writable 
data: concurrent threads happen to take an interest in the same piece of writable data at 
about the same time. It is not even necessary that the concurrent threads be running 
simultaneously; if one is stalled (perhaps because of an interrupt) in the middle of an 
action, a different, running thread can take an interest in the data that the stalled thread 
was, and will sometime again be, working with. 
From the point of view of the programmer of an application, Chapter 5 introduced 
two quite different kinds of concurrency coordination requirements: sequence coordina­
tion and before-or-after atomicity. Sequence coordination is a constraint of the type 
“Action W must happen before action X”. For correctness, the first action must complete 
before the second action begins. For example, reading of typed characters from a key­
board must happen before running the program that presents those characters on a 
display. As a general rule, when writing a program one can anticipate the sequence coor­
dination constraints, and the programmer knows the identity of the concurrent actions. 
Sequence coordination thus is usually explicitly programmed, using either special lan­
guage constructs or shared variables such as the eventcounts of Chapter 5. 
In contrast, before-or-after atomicity is a more general constraint that several actions 
that concurrently operate on the same data should not interfere with one another. We 
define before-or-after atomicity as follows: 
Before-or-after atomicity 
Concurrent actions have the before-or-after property if their effect from the point of 
view of their invokers is the same as if the actions occurred either completely before 
or completely after one another. 
In Chapter 5 we saw how before-or-after actions can be created with explicit locks and 
a thread manager that implements the procedures ACQUIRE and RELEASE. Chapter 5 showed 
some examples of before-or-after actions using locks, and emphasized that programming 
correct before-or-after actions, for example coordinating a bounded buffer with several 
producers or several consumers, can be a tricky proposition. To be confident of correct­
ness, one needs to establish a compelling argument that every action that touches a 
shared variable follows the locking protocol. 
Saltzer & Kaashoek Ch. 9, p. 13 
June 24, 2009 12:26 am

## Page 16

9–14 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
One thing that makes before-or-after atomicity different from sequence coordination 
is that the programmer of an action that must have the before-or-after property does not 
necessarily know the identities of all the other actions that might touch the shared vari­
able. This lack of knowledge can make it problematic to coordinate actions by explicit 
program steps. Instead, what the programmer needs is an automatic, implicit mechanism 
that ensures proper handling of every shared variable. This chapter will describe several 
such mechanisms. Put another way, correct coordination requires discipline in the way 
concurrent threads read and write shared data. 
Applications for before-or-after atomicity in a computer system abound. In an oper­
ating system, several concurrent threads may decide to use a shared printer at about the 
same time. It would not be useful for printed lines of different threads to be interleaved 
in the printed output. Moreover, it doesn’t really matter which thread gets to use the 
printer first; the primary consideration is that one use of the printer be complete before 
the next begins, so the requirement is to give each print job the before-or-after atomicity 
property. 
For a more detailed example, let us return to the banking application and the TRANSFER 
procedure. This time the account balances are held in shared memory variables (recall 
that the declaration keyword reference means that the argument is call-by-reference, so 
that TRANSFER can change the values of those arguments): 
procedure TRANSFER (reference debit_account, reference credit_account, amount) 
debit_account ← debit_account - amount 
credit_account ← credit_account + amount 
Despite their unitary appearance, a program statement such as “X ← X + Y” is actu­
ally composite: it involves reading the values of X and Y, performing an addition, and 
then writing the result back into X. If a concurrent thread reads and changes the value of 
X between the read and the write done by this statement, that other thread may be sur­
prised when this statement overwrites its change. 
Suppose this procedure is applied to accounts A (initially containing $300) and B (ini­
tially containing $100) as in 
TRANSFER (A, B, $10) 
We expect account A, the debit account, to end up with $290, and account B, the 
credit account, to end up with $110. Suppose, however, a second, concurrent thread is 
executing the statement 
TRANSFER (B, C, $25) 
where account C starts with $175. When both threads complete their transfers, we expect 
B to end up with $85 and C with $200. Further, this expectation should be fulfilled no 
matter which of the two transfers happens first. But the variable credit_account in the 
first thread is bound to the same object (account B) as the variable debit_account in the 
second thread. The risk to correctness occurs if the two transfers happen at about the 
same time. To understand this risk, consider Figure 9.2, which illustrates several possible 
time sequences of the READ and WRITE steps of the two threads with respect to variable B. 
Saltzer & Kaashoek Ch. 9, p. 14 
June 24, 2009 12:26 am

## Page 17

9.1 Atomicity 9–15  
With each time sequence the figure shows the history of values of the cell containing the 
balance of account B. If both steps 1–1 and 1–2 precede both steps 2–1 and 2–2, (or vice-
versa) the two transfers will work as anticipated, and B ends up with $85. If, however, 
step 2–1 occurs after step 1–1, but before step 1–2, a mistake will occur: one of the two 
transfers will not affect account B, even though it should have. The first two cases illus­
trate histories of shared variable B in which the answers are the correct result; the 
remaining four cases illustrate four different sequences that lead to two incorrect values 
for B. 
Thread #1 (credit_account is B) 
. . 
1–1 
1–2 
WRITE B 
READ B 
. 
Thread #2 (debit_account is B) 
. . 
2–1 
2–2 
WRITE B 
READ B 
. 
correct result:  
time 
case 1:  Thread #1: READ B 
WRITE B 
Thread #2: 
READ B 
WRITE B 
Value of B: 100 
110 
85 
case 2:  Thread #1: 
READ B 
WRITE B 
Thread #2: READ B 
WRITE B 
Value of B: 100 
75 
85 
wrong results: 
case 3:  Thread #1: READ B 
WRITE B 
Thread #2: 
READ B 
WRITE B 
Value of B: 100 
110 
75 
case 4:  Thread #1: READ B 
WRITE B 
Thread #2: 
READ B 
WRITE B 
Value of B: 100 
75 
110 
case 5:  Thread #1: 
READ B 
WRITE B 
Thread #2: READ B 
WRITE B 
Value of B: 100 
110 
75 
case 6:  Thread #1: 
READ B 
WRITE B 
Thread #2: READ B 
WRITE B 
Value of B: 100 
75 
110 
FIGURE 9.2 
Six possible histories of variable B if two threads that share B do not coordinate their concur­
rent activities. 
Saltzer & Kaashoek Ch. 9, p. 15  
June 24, 2009 12:26 am

## Page 18

9–16 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
Thus our goal is to ensure that one of the first two time sequences actually occurs. 
One way to achieve this goal is that the two steps 1–1 and 1–2 should be atomic, and the 
two steps 2–1 and 2–2 should similarly be atomic. In the original program, the steps 
debit_account ← debit_account - amount 
and 
credit_account ← credit_account + amount 
should each be atomic. There should be no possibility that a concurrent thread that 
intends to change the value of the shared variable debit_account read its value between 
the READ and WRITE steps of this statement. 
9.1.6 Correctness and Serialization 
The notion that the first two sequences of Figure 9.2 are correct and the other four are 
wrong is based on our understanding of the banking application. It would be better to 
have a more general concept of correctness that is independent of the application. Appli­
cation independence is a modularity goal: we want to be able to make an argument for 
correctness of the mechanism that provides before-or-after atomicity without getting 
into the question of whether or not the application using the mechanism is correct. 
There is such a correctness concept: coordination among concurrent actions can be 
considered to be correct if every result is guaranteed to be one that could have been obtained 
by some purely serial application of those same actions. 
The reasoning behind this concept of cor­
rectness involves several steps. Consider 
Figure 9.3,which shows, abstractly, the effect 
old system
state 
new system
state
 action 
of applying some action, whether atomic or 
not, to a system: the action changes the state 
of the system. Now, if we are sure that: 
1.  the old state of the system was correct 
FIGURE 9.3 
from the point of view of the 
application, and  
A single action takes a system from one 
state to another state. 
2.  the action, performing all by itself, 
correctly transforms any correct old state to a correct new state, 
then we can reason that the new state must also be correct. This line of reasoning holds 
for any application-dependent definition of “correct” and “correctly transform”, so our 
reasoning method is independent of those definitions and thus of the application. 
The corresponding requirement when several actions act concurrently, as in Figure 9.4, 
is that the resulting new state ought to be one of those that would have resulted from 
some serialization of the several actions, as in Figure 9.5. This correctness criterion means 
that concurrent actions are correctly coordinated if their result is guaranteed to be one 
that would have been obtained by some purely serial application of those same actions. 
Saltzer & Kaashoek Ch. 9, p. 16  
June 24, 2009 12:26 am

## Page 19

9.1 Atomicity 9–17  
FIGURE 9.4 
action #3 
action #1 
old system 
state 
new system 
state
action #2 
When several actions act con­
currently, they together 
produce a new state. If the 
actions are before-or-after and 
the old state was correct, the 
new state will be correct. 
So long as the only coordination requirement is before-or-after atomicity, any serializa­
tion will do. 
Moreover, we do not even need to insist that the system actually traverse the interme­
diate states along any particular path of Figure 9.5—it may instead follow the dotted 
trajectory through intermediate states that are not by themselves correct, according to the 
application’s definition. As long as the intermediate states are not visible above the 
implementing layer, and the system is guaranteed to end up in one of the acceptable final 
states, we can declare the coordination to be correct because there exists a trajectory that 
leads to that state for which a correctness argument could have been applied to every step. 
Since our definition of before-or-after atomicity is that each before-or-after action act 
as though it ran either completely before or completely after each other before-or-after 
action, before-or-after atomicity leads directly to this concept of correctness. Put another 
way, before-or-after atomicity has the effect of serializing the actions, so it follows that 
before-or-after atomicity guarantees correctness of coordination. A different way of 
old system 
state 
final 
state 
C 
final 
state 
B 
final 
state 
A
AA 
#1 
AA #2 
AA#3 
AA#3 
AA 
#2 
AA #2 
AA#3 
AA#1 
FIGURE 9.5 
We insist that the ﬁnal state be one that could have been reached by some serialization of the 
atomic actions, but we don't care which serialization. In addition, we do not need to insist that 
the intermediate states ever actually exist. The actual state trajectory could be that shown by 
the dotted lines, but only if there is no way of observing the intermediate states from the 
outside. 
Saltzer & Kaashoek Ch. 9, p. 17 
June 24, 2009 12:26 am

## Page 20

9–18 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
expressing this idea is to say that when concurrent actions have the before-or-after prop­
erty, they are serializable: there exists some serial order of those concurrent transactions that 
would, if followed, lead to the same ending state.* Thus in Figure 9.2, the sequences of case 
1 and case 2 could result from a serialized order, but the actions of cases 3 through 6 
could not. 
In the example of Figure 9.2, there were only two concurrent actions and each of the 
concurrent actions had only two steps. As the number of concurrent actions and the 
number of steps in each action grows there will be a rapidly growing number of possible 
orders in which the individual steps can occur, but only some of those orders will ensure 
a correct result. Since the purpose of concurrency is to gain performance, one would like 
to have a way of choosing from the set of correct orders the one correct order that has 
the highest performance. As one might guess, making that choice can in general be quite 
difficult. In Sections 9.4 and 9.5 of this chapter we will encounter several programming 
disciplines that ensure choice from a subset of the possible orders, all members of which 
are guaranteed to be correct but, unfortunately, may not include the correct order that 
has the highest performance. 
In some applications it is appropriate to use a correctness requirement that is stronger 
than serializability. For example, the designer of a banking system may want to avoid 
anachronisms by requiring what might be called external time consistency: if there is any 
external evidence (such as a printed receipt) that before-or-after action T1 ended before 
before-or-after action T2 began, the serialization order of T1 and T2 inside the system 
should be that T1 precedes T2. For another example of a stronger correctness require­
ment, a processor architect may require sequential consistency: when the processor 
concurrently performs multiple instructions from the same instruction stream, the result 
should be as if the instructions were executed in the original order specified by the 
programmer. 
Returning to our example, a real funds-transfer application typically has several dis­
tinct before-or-after atomicity requirements. Consider the following auditing procedure; 
its purpose is to verify that the sum of the balances of all accounts is zero (in double-entry 
bookkeeping, accounts belonging to the bank, such as the amount of cash in the vault, 
have negative balances): 
procedure AUDIT()  
sum ← 0  
for each W ← in bank.accounts  
sum ← sum + W.balance  
if (sum ≠ 0) call for investigation  
Suppose that AUDIT is running in one thread at the same time that another thread is 
transferring money from account A to account B. If AUDIT examines account A before the 
transfer and account B after the transfer, it will count the transferred amount twice and 
* The general question of whether or not a collection of existing transactions is serializable is an 
advanced topic that is addressed in database management. Problem set 36 explores one method of 
answering this question. 
Saltzer & Kaashoek Ch. 9, p. 18 
June 24, 2009 12:26 am

## Page 21

9.1 Atomicity 9–19  
thus will compute an incorrect answer. So the entire auditing procedure should occur 
either before or after any individual transfer: we want it to be a before-or-after action. 
There is yet another before-or-after atomicity requirement: if AUDIT should run after 
the statement in TRANSFER 
debit_account ← debit_account - amount 
but before the statement 
credit_account ← credit_account + amount 
it will calculate a sum that does not include amount; we therefore conclude that the two 
balance updates should occur either completely before or completely after any AUDIT 
action; put another way, TRANSFER should be a before-or-after action. 
9.1.7 All-or-Nothing and Before-or-After Atomicity 
We now have seen examples of two forms of atomicity: all-or-nothing and before-or­
after. These two forms have a common underlying goal: to hide the internal structure of 
an action. With that insight, it becomes apparent that atomicity is really a unifying 
concept: 
Atomicity 
An action is atomic if there is no way for a higher layer to discover the internal structure 
of its implementation. 
This description is really the fundamental definition of atomicity. From it, one can 
immediately draw two important consequences, corresponding to all-or-nothing atom­
icity and to before-or-after atomicity: 
1.  From the point of view of a procedure that invokes an atomic action, the atomic 
action always appears either to complete as anticipated, or to do nothing. This 
consequence is the one that makes atomic actions useful in recovering from 
failures. 
2.  From the point of view of a concurrent thread, an atomic action acts as though it 
occurs either completely before or completely after every other concurrent atomic 
action. This consequence is the one that makes atomic actions useful for 
coordinating concurrent threads. 
These two consequences are not fundamentally different. They are simply two per­
spectives, the first from other modules within the thread that invokes the action, the 
second from other threads. Both points of view follow from the single idea that the inter­
nal structure of the action is not visible outside of the module that implements the 
action. Such hiding of internal structure is the essence of modularity, but atomicity is an 
exceptionally strong form of modularity. Atomicity hides not just the details of which 
Saltzer & Kaashoek Ch. 9, p. 19  
June 24, 2009 12:26 am

## Page 22

9–20 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
steps form the atomic action, but the very fact that it has structure. There is a kinship 
between atomicity and other system-building techniques such as data abstraction and cli­
ent/server organization. Data abstraction has the goal of hiding the internal structure of 
data; client/server organization has the goal of hiding the internal structure of major sub­
systems. Similarly, atomicity has the goal of hiding the internal structure of an action. 
All three are methods of enforcing industrial-strength modularity, and thereby of guar­
anteeing absence of unanticipated interactions among components of a complex system. 
We have used phrases such as “from the point of view of the invoker” several times, 
suggesting that there may be another point of view from which internal structure is 
apparent. That other point of view is seen by the implementer of an atomic action, who 
is often painfully aware that an action is actually composite, and who must do extra work 
to hide this reality from the higher layer and from concurrent threads. Thus the inter­
faces between layers are an essential part of the definition of an atomic action, and they 
provide an opportunity for the implementation of an action to operate in any way that 
ends up providing atomicity. 
There is one more aspect of hiding the internal structure of atomic actions: atomic 
actions can have benevolent side effects. A common example is an audit log, where 
atomic actions that run into trouble record the nature of the detected failure and the 
recovery sequence for later analysis. One might think that when a failure leads to backing 
out, the audit log should be rolled back, too; but rolling it back would defeat its pur­
pose—the whole point of an audit log is to record details about the failure. The 
important point is that the audit log is normally a private record of the layer that imple­
mented the atomic action; in the normal course of operation it is not visible above that 
layer, so there is no requirement to roll it back. (A separate atomicity requirement is to 
ensure that the log entry that describes a failure is complete and not lost in the ensuing 
recovery.) 
Another example of a benevolent side effect is performance optimization. For exam­
ple, in a high-performance data management system, when an upper layer atomic action 
asks the data management system to insert a new record into a file, the data management 
system may decide as a performance optimization that now is the time to rearrange the 
file into a better physical order. If the atomic action fails and aborts, it need ensure only 
that the newly-inserted record be removed; the file does not need to be restored to its 
older, less efficient, storage arrangement. Similarly, a lower-layer cache that now contains 
a variable touched by the atomic action does not need to be cleared and a garbage collec­
tion of heap storage does not need to be undone. Such side effects are not a problem, as 
long as they are hidden from the higher-layer client of the atomic action except perhaps 
in the speed with which later actions are carried out, or across an interface that is 
intended to report performance measures or failures. 
Saltzer & Kaashoek Ch. 9, p. 20 
June 24, 2009 12:26 am

## Page 23

9.2 All-or-Nothing Atomicity I: Concepts 9–21  
9.2 All-or-Nothing Atomicity I: Concepts 
Section 9.1 of this chapter defined the goals of all-or-nothing atomicity and before-or­
after atomicity, and provided a conceptual framework that at least in principle allows a 
designer to decide whether or not some proposed algorithm correctly coordinates con­
current activities. However, it did not provide any examples of actual implementations 
of either goal. This section of the chapter, together with the next one, describe some 
widely applicable techniques of systematically implementing all-or-nothing atomicity. 
Later sections of the chapter will do the same for before-or-after atomicity. 
Many of the examples employ the technique introduced in Chapter 5 called boot­
strapping, a method that resembles inductive proof. To review, bootstrapping means to 
first look for a systematic way to reduce a general problem to some much-narrowed par­
ticular version of that same problem. Then, solve the narrow problem using some 
specialized method that might work only for that case because it takes advantage of the 
specific situation. The general solution then consists of two parts: a special-case tech­
nique plus a method that systematically reduces the general problem to the special case. 
Recall that Chapter 5 tackled the general problem of creating before-or-after actions 
from arbitrary sequences of code by implementing a procedure named ACQUIRE that itself 
required before-or-after atomicity of two or three lines of code where it reads and then 
sets a lock value. It then implemented that before-or-after action with the help of a spe­
cial hardware feature that directly makes a before-or-after action of the read and set 
sequence, and it also exhibited a software implementation (in Sidebar 5.2) that relies only 
on the hardware performing ordinary LOADs and STOREs as before-or-after actions. This 
chapter uses bootstrapping several times. The first example starts with the special case 
and then introduces a way to reduce the general problem to that special case. The reduc­
tion method, called the version history, is used only occasionally in practice, but once 
understood it becomes easy to see why the more widely used reduction methods that will 
be described in Section 9.3 work. 
9.2.1 Achieving All-or-Nothing Atomicity: ALL_OR_NOTHING_PUT 
The first example is of a scheme that does an all-or-nothing update of a single disk sector. 
The problem to be solved is that if a system crashes in the middle of a disk write (for 
example, the operating system encounters a bug or the power fails), the sector that was 
being written at the instant of the failure may contain an unusable muddle of old and 
new data. The goal is to create an all-or-nothing PUT with the property that when GET later 
reads the sector, it always returns either the old or the new data, but never a muddled 
mixture. 
To make the implementation precise, we develop a disk fault tolerance model that is 
a slight variation of the one introduced in Chapter 8[on-line], taking as an example 
application a calendar management program for a personal computer. The user is hoping 
that, if the system fails while adding a new event to the calendar, when the system later 
restarts the calendar will be safely intact. Whether or not the new event ended up in the 
Saltzer & Kaashoek Ch. 9, p. 21 
June 24, 2009 12:26 am

## Page 24

9–22 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
calendar is less important than that the calendar not be damaged by inopportune timing 
of the system failure. This system comprises a human user, a display, a processor, some 
volatile memory, a magnetic disk, an operating system, and the calendar manager pro­
gram. We model this system in several parts: 
Overall system fault tolerance model.  
•   error-free operation: All work goes according to expectations. The user initiates 
actions such as adding events to the calendar and the system confirms the actions 
by displaying messages to the user. 
•  tolerated error: The user who has initiated an action notices that the system 
failed before it confirmed completion of the action and, when the system is 
operating again, checks to see whether or not it actually performed that action. 
•   untolerated error: The system fails without the user noticing, so the user does 
not realize that he or she should check or retry an action that the system may not 
have completed. 
The tolerated error specification means that, to the extent possible, the entire system 
is fail-fast: if something goes wrong during an update, the system stops before taking any 
more requests, and the user realizes that the system has stopped. One would ordinarily 
design a system such as this one to minimize the chance of the untolerated error, for 
example by requiring supervision by a human user. The human user then is in a position 
to realize (perhaps from lack of response) that something has gone wrong. After the sys­
tem restarts, the user knows to inquire whether or not the action completed. This design 
strategy should be familiar from our study of best effort networks in Chapter 7[on-line]. 
The lower layer (the computer system) is providing a best effort implementation. A 
higher layer (the human user) supervises and, when necessary, retries. For example, sup­
pose that the human user adds an appointment to the calendar but just as he or she clicks 
“save” the system crashes. The user doesn’t know whether or not the addition actually 
succeeded, so when the system comes up again the first thing to do is open up the calen­
dar to find out what happened. 
Processor, memory, and operating system fault tolerance model.  
This part of the model just specifies more precisely the intended fail-fast properties of 
the hardware and operating system: 
•  error-free operation: The processor, memory, and operating system all follow 
their specifications. 
•   detected error: Something fails in the hardware or operating system. The system 
is fail-fast: the hardware or operating system detects the failure and restarts from 
a clean slate before initiating any further PUTs to the disk. 
•  untolerated error: Something fails in the hardware or operating system. The 
processor muddles along and PUTs corrupted data to the disk before detecting the 
failure. 
Saltzer & Kaashoek Ch. 9, p. 22  
June 24, 2009 12:26 am

## Page 25

9.2 All-or-Nothing Atomicity I: Concepts 9–23  
The primary goal of the processor/memory/operating-system part of the model is to 
detect failures and stop running before any corrupted data is written to the disk storage 
system. The importance of detecting failure before the next disk write lies in error con­
tainment: if the goal is met, the designer can assume that the only values potentially in 
error must be in processor registers and volatile memory, and the data on the disk should 
be safe, with the exception described in Section 8.5.4.2: if there was a PUT to the disk in 
progress at the time of the crash, the failing system may have corrupted the disk buffer 
in volatile memory, and consequently corrupted the disk sector that was being written. 
The recovery procedure can thus depend on the disk storage system to contain only 
uncorrupted information, or at most one corrupted disk sector. In fact, after restart the 
disk will contain the only information. “Restarts from a clean slate” means that the sys­
tem discards all state held in volatile memory. This step brings the system to the same 
state as if a power failure had occurred, so a single recovery procedure will be able to han­
dle both system crashes and power failures. Discarding volatile memory also means that 
all currently active threads vanish, so everything that was going on comes to an abrupt 
halt and will have to be restarted. 
Disk storage system fault tolerance model.  
Implementing all-or-nothing atomicity involves some steps that resemble the decay 
masking of MORE_DURABLE_PUT/GET in Chapter 8[on-line]—in particular, the algorithm 
will write multiple copies of data. To clarify how the all-or-nothing mechanism works, 
we temporarily back up to CAREFUL_PUT/GET (see Section 8.5.4.5), which masks soft disk 
errors but not hard disk errors or disk decay. To simplify further, we pretend for the 
moment that a disk never decays and that it has no hard errors. (Since this perfect-disk 
assumption is obviously unrealistic, we will reverse it in Section 9.7, which describes an 
algorithm for all-or-nothing atomicity despite disk decay and hard errors.) 
With the perfect-disk assumption, only one thing can go wrong: a system crash at 
just the wrong time. The fault tolerance model for this simplified careful disk system 
then becomes: 
•   error-free operation: CAREFUL_GET returns the result of the most recent call to 
CAREFUL_PUT at sector_number on track, with status = OK. 
•   detectable error: The operating system crashes during a CAREFUL_PUT and corrupts 
the disk buffer in volatile storage, and CAREFUL_PUT writes corrupted data on one 
sector of the disk. 
We can classify the error as “detectable” if we assume that the application has 
included with the data an end-to-end checksum, calculated before calling CAREFUL_PUT 
and thus before the system crash could have corrupted the data. 
The change in this revision of the careful storage layer is that when a system crash 
occurs, one sector on the disk may be corrupted, but the client of the interface is confi­
dent that (1) that sector is the only one that may be corrupted and (2) if it has been 
corrupted, any later reader of that sector will detect the problem. Between the processor 
model and the storage system model, all anticipated failures now lead to the same situa-
Saltzer & Kaashoek Ch. 9, p. 23  
June 24, 2009 12:26 am

## Page 26

9–24 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.6 
1 procedure ALMOST_ALL_OR_NOTHING_PUT (data, all_or_nothing_sector) 
2 
CAREFUL_PUT (data, all_or_nothing_sector.S1) 
3 
CAREFUL_PUT (data, all_or_nothing_sector.S2) 
// Commit point. 
4 
CAREFUL_PUT (data, all_or_nothing_sector.S3) 
5 procedure ALL_OR_NOTHING_GET (reference data, all_or_nothing_sector) 
6 
CAREFUL_GET (data1, all_or_nothing_sector.S1) 
7 
CAREFUL_GET (data2, all_or_nothing_sector.S2) 
8 
CAREFUL_GET (data3, all_or_nothing_sector.S3) 
9 
if data1 = data2 then data ← data1 
// Return new value. 
10 
else data ← data3 
// Return old value. 
Algorithms for ALMOST_ALL_OR_NOTHING_PUT and ALL_OR_NOTHING_GET. 
tion: the system detects the failure, resets all processor registers and volatile memory, 
forgets all active threads, and restarts. No more than one disk sector is corrupted. 
Our problem is now reduced to providing the all-or-nothing property: the goal is to 
create all-or-nothing disk storage, which guarantees either to change the data on a sector 
completely and correctly or else appear to future readers not to have touched it at all. 
Here is one simple, but somewhat inefficient, scheme that makes use of virtualization: 
assign, for each data sector that is to have the all-or-nothing property, three physical disk 
sectors, identified as S1, S2, and S3. The three physical sectors taken together are a vir­
tual “all-or-nothing sector”. At each place in the system where this disk sector was 
previously used, replace it with the all-or-nothing sector, identified by the triple {S1, S2, 
S3}. We start with an almost correct all-or-nothing implementation named 
ALMOST_ALL_OR_NOTHING_PUT, find a bug in it, and then fix the bug, finally creating a cor­
rect ALL_OR_NOTHING_PUT. 
When asked to write data, ALMOST_ALL_OR_NOTHING_PUT writes it three times, on S1, S2, 
and S3, in that order, each time waiting until the previous write finishes, so that if the 
system crashes only one of the three sectors will be affected. To read data, 
ALL_OR_NOTHING_GET reads all three sectors and compares their contents. If the contents of 
S1 and S2 are identical, ALL_OR_NOTHING_GET returns that value as the value of the all-or­
nothing sector. If S1 and S2 differ, ALL_OR_NOTHING_GET returns the contents of S3 as the 
value of the all-or-nothing sector. Figure 9.6 shows this almost correct pseudocode. 
Let’s explore how this implementation behaves on a system crash. Suppose that at 
some previous time a record has been correctly stored in an all-or-nothing sector (in 
other words, all three copies are identical), and someone now updates it by calling 
ALL_OR_NOTHING_PUT. The goal is that even if a failure occurs in the middle of the update, 
a later reader can always be ensured of getting some complete, consistent version of the 
record by invoking ALL_OR_NOTHING_GET. 
Suppose that ALMOST_ALL_OR_NOTHING_PUT were interrupted by a system crash some 
time before it finishes writing sector S2, and thus corrupts either S1 or S2. In that case, 
Saltzer & Kaashoek Ch. 9, p. 24 
June 24, 2009 12:26 am

## Page 27

9.2 All-or-Nothing Atomicity I: Concepts 9–25  
FIGURE 9.7 
1 procedure ALL_OR_NOTHING_PUT (data, all_or_nothing_sector) 
2 
CHECK_AND_REPAIR (all_or_nothing_sector) 
3 
ALMOST_ALL_OR_NOTHING_PUT (data, all_or_nothing_sector) 
4 procedure CHECK_AND_REPAIR (all_or_nothing_sector) // Ensure copies match. 
5 
CAREFUL_GET (data1, all_or_nothing_sector.S1) 
6 
CAREFUL_GET (data2, all_or_nothing_sector.S2) 
7 
CAREFUL_GET (data3, all_or_nothing_sector.S3) 
8 
if (data1 = data2) and (data2 = data3) return 
// State 1 or 7, no repair 
9 
if (data1 = data2) 
10 
CAREFUL_PUT (data1, all_or_nothing_sector.S3) return 
// State 5 or 6. 
11 
if (data2 = data3) 
12 
CAREFUL_PUT (data2, all_or_nothing_sector.S1) return 
// State 2 or 3. 
13 
CAREFUL_PUT (data1, all_or_nothing_sector.S2) 
// State 4, go to state 5 
14 
CAREFUL_PUT (data1, all_or_nothing_sector.S3) 
// State 5, go to state 7 
Algorithms for ALL_OR_NOTHING_PUT and CHECK_AND_REPAIR. 
when ALL_OR_NOTHING_GET reads sectors S1 and S2, they will have different values, and it 
is not clear which one to trust. Because the system is fail-fast, sector S3 would not yet 
have been touched by ALMOST_ALL_OR_NOTHING_PUT, so it still contains the previous value. 
Returning the value found in S3 thus has the desired effect of ALMOST_ALL_OR_NOTHING_PUT 
having done nothing. 
Now, suppose that ALMOST_ALL_OR_NOTHING_PUT were interrupted by a system crash 
some time after successfully writing sector S2. In that case, the crash may have corrupted 
S3, but S1 and S2 both contain the newly updated value. ALL_OR_NOTHING_GET returns the 
value of S1, thus providing the desired effect of ALMOST_ALL_OR_NOTHING_PUT having com­
pleted its job. 
So what’s wrong with this design? ALMOST_ALL_OR_NOTHING_PUT assumes that all three 
copies are identical when it starts. But a previous failure can violate that assumption. 
Suppose that ALMOST_ALL_OR_NOTHING_PUT is interrupted while writing S3. The next 
thread to call ALL_OR_NOTHING_GET finds data1 = data2, so it uses data1, as expected. The 
new thread then calls ALMOST_ALL_OR_NOTHING_PUT, but is interrupted while writing S2. 
Now, S1 doesn't equal S2, so the next call to ALMOST_ALL_OR_NOTHING_PUT returns the 
damaged S3. 
The fix for this bug is for ALL_OR_NOTHING_PUT to guarantee that the three sectors be iden­
tical before updating. It can provide this guarantee by invoking a procedure named 
CHECK_AND_REPAIR as in Figure 9.7. CHECK_AND_REPAIR simply compares the three copies 
and, if they are not identical, it forces them to be identical. To see how this works, assume 
that someone calls ALL_OR_NOTHING_PUT at a time when all three of the copies do contain 
identical values, which we designate as “old”. Because ALL_OR_NOTHING_PUT writes “new” 
Saltzer & Kaashoek Ch. 9, p. 25 
June 24, 2009 12:26 am

## Page 28

9–26 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
values into S1, S2, and S3 one at a time and in order, even if there is a crash, at the next 
call to ALL_OR_NOTHING_PUT there are only seven possible data states for CHECK_AND_REPAIR 
to consider: 
data state: 
1 
2
 3
 4
 5
 6
 7 
sector S1 
old 
bad 
new 
new 
new 
new 
new 
sector S2 
old 
old 
old 
bad 
new 
new 
new 
sector S3 
old 
old 
old 
old 
old 
bad 
new 
The way to read this table is as follows: if all three sectors S1, S2, and S3 contain the 
“old” value, the data is in state 1. Now, if CHECK_AND_REPAIR discovers that all three copies 
are identical (line 8 in Figure 9.7), the data is in state 1 or state 7 so CHECK_AND_REPAIR 
simply returns. Failing that test, if the copies in sectors S1 and S2 are identical (line 9), 
the data must be in state 5 or state 6, so CHECK_AND_REPAIR forces sector S3 to match and 
returns (line 10). If the copies in sectors S2 and S3 are identical the data must be in state 
2 or state 3 (line 11), so CHECK_AND_REPAIR forces sector S1 to match and returns (line 12). 
The only remaining possibility is that the data is in state 4, in which case sector S2 is 
surely bad, but sector S1 contains a new value and sector S3 contains an old one. The 
choice of which to use is arbitrary; as shown the procedure copies the new value in sector 
S1 to both sectors S2 and S3. 
What if a failure occurs while running CHECK_AND_REPAIR? That procedure systemati­
cally drives the state either forward from state 4 toward state 7, or backward from state 
3 toward state 1. If CHECK_AND_REPAIR is itself interrupted by another system crash, rerun­
ning it will continue from the point at which the previous attempt left off. 
We can make several observations about the algorithm implemented by 
ALL_OR_NOTHING_GET and ALL_OR_NOTHING_PUT: 
1.  This all-or-nothing atomicity algorithm assumes that only one thread at a time 
tries to execute either ALL_OR_NOTHING_GET or ALL_OR_NOTHING_PUT. This algorithm 
implements all-or-nothing atomicity but not before-or-after atomicity. 
2.  CHECK_AND_REPAIR is idempotent. That means that a thread can start the procedure, 
execute any number of its steps, be interrupted by a crash, and go back to the 
beginning again any number of times with the same ultimate result, as far as a later 
call to ALL_OR_NOTHING_GET is concerned. 
3.  The completion of the CAREFUL_PUT on line 3 of ALMOST_ALL_OR_NOTHING_PUT, 
marked “commit point,” exposes the new data to future ALL_OR_NOTHING_GET 
actions. Until that step begins execution, a call to ALL_OR_NOTHING_GET sees the old 
data. After line 3 completes, a call to ALL_OR_NOTHING_GET sees the new data. 
4.  Although the algorithm writes three replicas of the data, the primary reason for 
the replicas is not to provide durability as described in Section 8.5. Instead, the 
reason for writing three replicas, one at a time and in a particular order, is to ensure 
observance at all times and under all failure scenarios of the golden rule of atomicity, 
which is the subject of the next section. 
Saltzer & Kaashoek Ch. 9, p. 26  
June 24, 2009 12:26 am

## Page 29

9.2 All-or-Nothing Atomicity I: Concepts 9–27  
There are several ways of implementing all-or-nothing disk sectors. Near the end of 
Chapter 8[on-line] we introduced a fault tolerance model for decay events that did not 
mask system crashes, and applied the technique known as RAID to mask decay to pro­
duce durable storage. Here we started with a slightly different fault tolerance model that 
omits decay, and we devised techniques to mask system crashes and produce all-or-noth­
ing storage. What we really should do is start with a fault tolerance model that considers 
both system crashes and decay, and devise storage that is both all-or-nothing and dura­
ble. Such a model, devised by Xerox Corporation researchers Butler Lampson and 
Howard Sturgis, is the subject of Section 9.7, together with the more elaborate recovery 
algorithms it requires. That model has the additional feature that it needs only two phys­
ical sectors for each all-or-nothing sector. 
9.2.2 Systematic Atomicity: Commit and the Golden Rule 
The example of ALL_OR_NOTHING_PUT and ALL_OR_NOTHING_GET demonstrates an interesting 
special case of all-or-nothing atomicity, but it offers little guidance on how to systemat­
ically create a more general all-or-nothing action. From the example, our calendar 
program now has a tool that allows writing individual sectors with the all-or-nothing 
property, but that is not the same as safely adding an event to a calendar, since adding 
an event probably requires rearranging a data structure, which in turn may involve writ­
ing more than one disk sector. We could do a series of ALL_OR_NOTHING_PUTs to the several 
sectors, to ensure that each sector is itself written in an all-or-nothing fashion, but a crash 
that occurs after writing one and before writing the next would leave the overall calendar 
addition in a partly-done state. To make the entire calendar addition action all-or-noth­
ing we need a generalization. 
Ideally, one might like to be able to take any arbitrary sequence of instructions in a 
program, surround that sequence with some sort of begin and end statements as in Fig­
ure 9.8, and expect that the language compilers and operating system will perform some 
magic that makes the surrounded sequence into an all-or-nothing action. Unfortunately, 
no one knows how to do that. But we can come close, if the programmer is willing to 
make a modest concession to the requirements of all-or-nothing atomicity. This conces­
sion is expressed in the form of a discipline on the constituent steps of the all-or-nothing 
action. 
The discipline starts by identifying some single step of the sequence as the commit 
point. The all-or-nothing action is thus divided into two phases, a pre-commit phase and 
a post-commit phase, as suggested by Figure 9.9. During the pre-commit phase, the disci­
plining rule of design is that no matter what happens, it must be possible to back out of 
this all-or-nothing action in a way that leaves no trace. During the post-commit phase 
the disciplining rule of design is that no matter what happens, the action must run to the 
end successfully. Thus an all-or-nothing action can have only two outcomes. If the all-
or-nothing action starts and then, without reaching the commit point, backs out, we say 
that it aborts. If the all-or-nothing action passes the commit point, we say that it commits. 
Saltzer & Kaashoek Ch. 9, p. 27 
June 24, 2009 12:26 am

## Page 30

9–28 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.8 
___ 
___ 
___ 
begin all-or-nothing action 
___ 
___ 
___ 
arbitrary sequence of 
___ 
lower-layer actions 
___ 
end all-or-nothing action 
___ 
___ 
___ 
} 
Imaginary semantics for painless programming of all-or-nothing actions. 
We can make several observations about the restrictions of the pre-commit phase. 
The pre-commit phase must identify all the resources needed to complete the all-or­
nothing action, and establish their availability. The names of data should be bound, per­
missions should be checked, the pages to be read or written should be in memory, 
removable media should be mounted, stack space must be allocated, etc. In other words, 
all the steps needed to anticipate the severe run-to-the-end-without-faltering require­
ment of the post-commit phase should be completed during the pre-commit phase. In 
addition, the pre-commit phase must maintain the ability to abort at any instant. Any 
changes that the pre-commit phase makes to the state of the system must be undoable in 
case this all-or-nothing action aborts. Usually, this requirement means that shared 
___ 
first step of all-or-nothing action 
___ 
___ 
___ 
Pre-commit discipline: can back out, 
___ 
leaving no trace 
___ 
___ 
___ 
Commit point 
___ 
___ 
___ 
Post-commit discipline: completion is inevitable 
___ 
___ 
last step of all-or-nothing action
} 
} 
FIGURE 9.9 
The commit point of an all-or-nothing action. 
Saltzer & Kaashoek Ch. 9, p. 28 
June 24, 2009 12:26 am

## Page 31

9.2 All-or-Nothing Atomicity I: Concepts 9–29  
resources, once reserved, cannot be released until the commit point is passed. The reason 
is that if an all-or-nothing action releases a shared resource, some other, concurrent 
thread may capture that resource. If the resource is needed in order to undo some effect 
of the all-or-nothing action, releasing the resource is tantamount to abandoning the abil­
ity to abort. Finally, the reversibility requirement means that the all-or-nothing action 
should not do anything externally visible, for example printing a check or firing a missile, 
prior to the commit point. (It is possible, though more complicated, to be slightly less 
restrictive. Sidebar 9.3 explores that possibility.) 
In contrast, the post-commit phase can expose results, it can release reserved resources 
that are no longer needed, and it can perform externally visible actions such as printing 
a check, opening a cash drawer, or drilling a hole. But it cannot try to acquire additional 
resources because an attempt to acquire might fail, and the post-commit phase is not per­
mitted the luxury of failure. The post-commit phase must confine itself to finishing just 
the activities that were planned during the pre-commit phase. 
It might appear that if a system fails before the post-commit phase completes, all hope 
is lost, so the only way to ensure all-or-nothing atomicity is to always make the commit 
step the last step of the all-or-nothing action. Often, that is the simplest way to ensure 
all-or-nothing atomicity, but the requirement is not actually that stringent. An impor­
tant feature of the post-commit phase is that it is hidden inside the layer that implements 
the all-or-nothing action, so a scheme that ensures that the post-commit phase completes 
after a system failure is acceptable, so long as this delay is hidden from the invoking layer. 
Some all-or-nothing atomicity schemes thus involve a guarantee that a cleanup proce­
dure will be invoked following every system failure, or as a prelude to the next use of the 
data, before anyone in a higher layer gets a chance to discover that anything went wrong. 
This idea should sound familiar: the implementation of ALL_OR_NOTHING_PUT in Figure 
9.7 used this approach, by always running the cleanup procedure named 
CHECK_AND_REPAIR before updating the data. 
A popular technique for achieving all-or-nothing atomicity is called the shadow copy. 
It is used by text editors, compilers, calendar management programs, and other programs 
that modify existing files, to ensure that following a system failure the user does not end 
up with data that is damaged or that contains only some of the intended changes: 
•   Pre-commit: Create a complete duplicate working copy of the file that is to be 
modified. Then, make all changes to the working copy. 
Sidebar 9.3:  Cascaded aborts  (Temporary) sweeping simpliﬁcation. In this initial discussioin of 
commit points, we are intentionally avoiding a more complex and harder-to-design possibility. 
Some systems allow other, concurrent activities to see pending results, and they may even allow 
externally visible actions before commit. Those systems must therefore be prepared to track 
down and abort those concurrent activities (this tracking down is called cascaded abort) or 
perform compensating external actions (e.g., send a letter requesting return of the check or 
apologizing for the missile ﬁring). The discussion of layers and multiple sites in Chapter 10[on­
line] introduces a simple version of cascaded abort. 
Saltzer & Kaashoek Ch. 9, p. 29  
June 24, 2009 12:26 am

## Page 32

9–30 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
•   Commit point: Carefully exchange the working copy with the original. Typically 
this step is bootstrapped, using a lower-layer RENAME entry point of the file system 
that provides certain atomic-like guarantees such as the ones described for the 
UNIX version of RENAME in Section 2.5.8. 
•   Post-commit: Release the space that was occupied by the original. 
The ALL_OR_NOTHING_PUT algorithm of Figure 9.7 can be seen as a particular example 
of the shadow copy strategy, which itself is a particular example of the general pre-com­
mit/post-commit discipline. The commit point occurs at the instant when the new value 
of S2 is successfully written to the disk. During the pre-commit phase, while 
ALL_OR_NOTHING_PUT is checking over the three sectors and writing the shadow copy S1, a 
crash will leave no trace of that activity (that is, no trace that can be discovered by a later 
caller of ALL_OR_NOTHING_GET). The post-commit phase of ALL_OR_NOTHING_PUT consists of 
writing S3. 
From these examples we can extract an important design principle: 
The golden rule of atomicity 
Never modify the only copy! 
In order for a composite action to be all-or-nothing, there must be some way of reversing 
the effect of each of its pre-commit phase component actions, so that if the action does 
not commit it is possible to back out. As we continue to explore implementations of all-
or-nothing atomicity, we will notice that correct implementations always reduce at the 
end to making a shadow copy. The reason is that structure ensures that the implemen­
tation follows the golden rule. 
9.2.3 Systematic All-or-Nothing Atomicity: Version Histories 
This section develops a scheme to provide all-or-nothing atomicity in the general case of 
a program that modifies arbitrary data structures. It will be easy to see why the scheme 
is correct, but the mechanics can interfere with performance. Section 9.3 of this chapter 
then introduces a variation on the scheme that requires more thought to see why it is cor­
rect, but that allows higher-performance implementations. As before, we concentrate for 
the moment on all-or-nothing atomicity. While some aspects of before-or-after atomic­
ity will also emerge, we leave a systematic treatment of that topic for discussion in 
Sections 9.4 and 9.5 of this chapter. Thus the model to keep in mind in this section is 
that only a single thread is running. If the system crashes, after a restart the original 
thread is gone—recall from Chapter 8[on-line] the sweeping simpliﬁcation that threads 
are included in the volatile state that is lost on a crash and only durable state survives. 
After the crash, a new, different thread comes along and attempts to look at the data. The 
goal is that the new thread should always find that the all-or-nothing action that was in 
progress at the time of the crash either never started or completed successfully. 
Saltzer & Kaashoek Ch. 9, p. 30  
June 24, 2009 12:26 am

## Page 33

9.2 All-or-Nothing Atomicity I: Concepts 9–31  
In looking at the general case, a fundamental difficulty emerges: random-access mem­
ory and disk usually appear to the programmer as a set of named, shared, and rewritable 
storage cells, called cell storage. Cell storage has semantics that are actually quite hard to 
make all-or-nothing because the act of storing destroys old data, thus potentially violat­
ing the golden rule of atomicity. If the all-or-nothing action later aborts, the old value is 
irretrievably gone; at best it can only be reconstructed from information kept elsewhere. 
In addition, storing data reveals it to the view of later threads, whether or not the all-or­
nothing action that stored the value reached its commit point. If the all-or-nothing 
action happens to have exactly one output value, then writing that value into cell storage 
can be the mechanism of committing, and there is no problem. But if the result is sup­
posed to consist of several output values, all of which should be exposed simultaneously, 
it is harder to see how to construct the all-or-nothing action. Once the first output value 
is stored, the computation of the remaining outputs has to be successful; there is no going 
back. If the system fails and we have not been careful, a later thread may see some old 
and some new values. 
These limitations of cell storage did not plague the shopkeepers of Padua, who in the 
14th century invented double-entry bookkeeping. Their storage medium was leaves of 
paper in bound books and they made new entries with quill pens. They never erased or 
even crossed out entries that were in error; when they made a mistake they made another 
entry that reversed the mistake, thus leaving a complete history of their actions, errors, 
and corrections in the book. It wasn’t until the 1950’s, when programmers began to 
automate bookkeeping systems, that the notion of overwriting data emerged. Up until 
that time, if a bookkeeper collapsed and died while making an entry, it was always pos­
sible for someone else to seamlessly take over the books. This observation about the 
robustness of paper systems suggests that there is a form of the golden rule of atomicity 
that might allow one to be systematic: never erase anything. 
Examining the shadow copy technique used by the text editor provides a second use­
ful idea. The essence of the mechanism that allows a text editor to make several changes 
to a file, yet not reveal any of the changes until it is ready, is this: the only way another 
prospective reader of a file can reach it is by name. Until commit time the editor works 
on a copy of the file that is either not yet named or has a unique name not known outside 
the thread, so the modified copy is effectively invisible. Renaming the new version is the 
step that makes the entire set of updates simultaneously visible to later readers. 
These two observations suggest that all-or-nothing actions would be better served by 
a model of storage that behaves differently from cell storage: instead of a model in which 
a store operation overwrites old data, we instead create a new, tentative version of the 
data, such that the tentative version remains invisible to any reader outside this all-or­
nothing action until the action commits. We can provide such semantics, even though 
we start with traditional cell memory, by interposing a layer between the cell storage and 
the program that reads and writes data. This layer implements what is known as journal 
storage. The basic idea of journal storage is straightforward: we associate with every 
named variable not a single cell, but a list of cells in non-volatile storage; the values in 
the list represent the history of the variable. Figure 9.10 illustrates. Whenever any action 
Saltzer & Kaashoek Ch. 9, p. 31 
June 24, 2009 12:26 am

## Page 34

9–32 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
History of earlier versions 
Tentative 
next version 
16
14
112
29
5
20
7
Variable A: 
Current version 
FIGURE 9.10 
Version history of a variable in journal storage. 
proposes to write a new value into the variable, the journal storage manager appends the 
prospective new value to the end of the list. Clearly this approach, being history-preserv­
ing, offers some hope of being helpful because if an all-or-nothing action aborts, one can 
imagine a systematic way to locate and discard all of the new versions it wrote. Moreover, 
we can tell the journal storage manager to expect to receive tentative values, but to ignore 
them unless the all-or-nothing action that created them commits. The basic mechanism 
to accomplish such an expectation is quite simple; the journal storage manager should 
make a note, next to each new version, of the identity of the all-or-nothing action that 
created it. Then, at any later time, it can discover the status of the tentative version by 
inquiring whether or not the all-or-nothing action ever committed. 
Figure 9.11 illustrates the overall structure of such a journal storage system, imple­
mented as a layer that hides a cell storage system. (To reduce clutter, this journal storage 
system omits calls to create new and delete old variables.) In this particular model, we 
assign to the journal storage manager most of the job of providing tools for programming 
all-or-nothing actions. Thus the implementer of a prospective all-or-nothing action 
should begin that action by invoking the journal storage manager entry NEW_ACTION, and 
later complete the action by invoking either COMMIT or ABORT. If, in addition, actions per­
form all reads and writes of data by invoking the journal storage manager’s 
READ_CURRENT_VALUE and WRITE_NEW_VALUE entries, our hope is that the result will auto­
matically be all-or-nothing with no further concern of the implementer. 
How could this automatic all-or-nothing atomicity work? The first step is that the 
journal storage manager, when called at NEW_ACTION, should assign a nonce identifier to 
the prospective all-or-nothing action, and create, in non-volatile cell storage, a record of 
this new identifier and the state of the new all-or-nothing action. This record is called an 
outcome record; it begins its existence in the state PENDING; depending on the outcome it 
should eventually move to one of the states COMMITTED or ABORTED, as suggested by Figure 
9.12. No other state transitions are possible, except to discard the outcome record once 
Saltzer & Kaashoek Ch. 9, p. 32 
June 24, 2009 12:26 am

## Page 35

9.2 All-or-Nothing Atomicity I: Concepts 9–33  
NEW_ACTION 
READ_CURRENT_VALUE 
WRITE_NEW_VALUE 
COMMIT 
ABORT 
All-or-nothing Journal Storage System 
Cell Storage 
System 
– catalogs 
– versions 
– outcome 
records 
Journal 
READ 
WRITE 
ALLOCATE 
DEALLOCATE 
Storage 
Manager 
FIGURE 9.11 
Interface to and internal organization of an all-or-nothing storage system based on version his­
tories and journal storage. 
new all-or-nothing
action is 
created 
aborted 
all-or-nothing 
all-or-nothing 
action 
commits 
committed 
pending 
non-existent 
discarded 
action 
outcome record 
aborts 
state no longer 
of any interest 
FIGURE 9.12 
The allowed state transitions of an outcome record. 
Saltzer & Kaashoek Ch. 9, p. 33 
June 24, 2009 12:26 am

## Page 36

9–34 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.13 
1 procedure NEW_ACTION () 
2
 id
 
← NEW_OUTCOME_RECORD () 
3 
id.outcome_record.state ← PENDING 
4 
return id 
5 procedure COMMIT (reference id) 
6 
id.outcome_record.state ← COMMITTED 
7 procedure ABORT (reference id) 
8 
id.outcome_record.state ← ABORTED 
The procedures NEW_ACTION, COMMIT, and ABORT. 
there is no further interest in its state. Figure 9.13 illustrates implementations of the three 
procedures NEW_ACTION, COMMIT, and ABORT. 
When an all-or-nothing action calls the journal storage manager to write a new ver­
sion of some data object, that action supplies the identifier of the data object, a tentative 
new value for the new version, and the identifier of the all-or-nothing action. The journal 
storage manager calls on the lower-level storage management system to allocate in non­
volatile cell storage enough space to contain the new version; it places in the newly allo­
cated cell storage the new data value and the identifier of the all-or-nothing action. Thus 
the journal storage manager creates a version history as illustrated in Figure 9.14. Now, 
7 
03 
outcome 
records 
Object A 
pending
1794:
aborted
1423:
1101: 
committed 
1101 
1423 
1794
all-or-nothing 
action id: 
75
15
24
value: 
FIGURE 9.14 
Portion of a version history, with outcome records. Some thread has recently called 
WRITE_NEW_VALUE specifying data_id = A, new_value = 75, and client_id = 1794. A caller to 
READ_CURRENT_VALUE will read the value 24 for A. 
Saltzer & Kaashoek Ch. 9, p. 34 
June 24, 2009 12:26 am

## Page 37

9.2 All-or-Nothing Atomicity I: Concepts 9–35  
FIGURE 9.15 
1 procedure READ_CURRENT_VALUE (data_id, caller_id) 
2 
starting at end of data_id repeat until beginning 
3 
v ← previous version of data_id 
// Get next older version 
4 
a ← v.action_id // Identify the action a that created it 
5 
s ← a.outcome_record.state 
// Check action a’s outcome record 
6 
if s = COMMITTED then 
7 
return v.value 
8 
else skip v 
// Continue backward search 
9 
signal (“Tried to read an uninitialized variable!”) 
10 procedure WRITE_NEW_VALUE (reference data_id, new_value, caller_id) 
11 
if caller_id.outcome_record.state = PENDING 
12 
append new version v to data_id 
13 
v.value ← new_value 
14 
v.action_id ← caller_id 
else signal (“Tried to write outside of an all-or-nothing action!”) 
Algorithms followed by READ_CURRENT_VALUE and WRITE_NEW_VALUE. The parameter caller_id is 
the action identiﬁer returned by NEW_ACTION. In this version, only WRITE_NEW_VALUE uses 
caller_id. Later, READ_CURRENT_VALUE will also use it. 
when someone proposes to read a data value by calling READ_CURRENT_VALUE, the journal 
storage manager can review the version history, starting with the latest version and return 
the value in the most recent committed version. By inspecting the outcome records, the 
journal storage manager can ignore those versions that were written by all-or-nothing 
actions that aborted or that never committed. 
The procedures READ_CURRENT_VALUE and WRITE_NEW_VALUE thus follow the algorithms 
of Figure 9.15. The important property of this pair of algorithms is that if the current 
all-or-nothing action is somehow derailed before it reaches its call to COMMIT, the new ver­
sion it has created is invisible to invokers of READ_CURRENT_VALUE. (They are also invisible 
to the all-or-nothing action that wrote them. Since it is sometimes convenient for an all-
or-nothing action to read something that it has tentatively written, a different procedure, 
named READ_MY_PENDING_VALUE, identical to READ_CURRENT_VALUE except for a different test 
on line 6, could do that.) Moreover if, for example, all-or-nothing action 99 crashes 
while partway through changing the values of nineteen different data objects, all nine­
teen changes would be invisible to later invokers of READ_CURRENT_VALUE. If all-or-nothing 
action 99 does reach its call to COMMIT, that call commits the entire set of changes simul­
taneously and atomically, at the instant that it changes the outcome record from PENDING 
to COMMITTED. Pending versions would also be invisible to any concurrent action that 
reads data with READ_CURRENT_VALUE, a feature that will prove useful when we introduce 
concurrent threads and discuss before-or-after atomicity, but for the moment our only 
Saltzer & Kaashoek Ch. 9, p. 35 
June 24, 2009 12:26 am

## Page 38

9–36 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.16 
1 procedure TRANSFER (reference debit_account, reference credit_account, 
amount) 
2 
my_id ← NEW_ACTION () 
3 
xvalue ← READ_CURRENT_VALUE (debit_account, my_id) 
4 
xvalue ← xvalue - amount 
5 
WRITE_NEW_VALUE (debit_account, xvalue, my_id) 
6 
yvalue ← READ_CURRENT_VALUE (credit_account, my_id) 
7 
yvalue ← yvalue + amount 
8 
WRITE_NEW_VALUE (credit_account, yvalue, my_id) 
9 
if xvalue > 0 then 
10 
COMMIT (my_id) 
11 
else 
12 
ABORT (my_id) 
13 
signal(“Negative transfers are not allowed.”) 
An all-or-nothing TRANSFER procedure, based on journal storage. (This program assumes that 
it is the only running thread. Making the transfer procedure a before-or-after action because 
other threads might be updating the same accounts concurrently requires additional mecha­
nism that is discussed later in this chapter.) 
concern is that a system crash may prevent the current thread from committing or abort­
ing, and we want to make sure that a later thread doesn’t encounter partial results. As in 
the case of the calendar manager of Section 9.2.1, we assume that when a crash occurs, 
any all-or-nothing action that was in progress at the time was being supervised by some 
outside agent who realizes that a crash has occurred, uses READ_CURRENT_VALUE to find out 
what happened and if necessary initiates a replacement all-or-nothing action. 
Figure 9.16 shows the TRANSFER procedure of Section 9.1.5 reprogrammed as an all-
or-nothing (but not, for the moment, before-or-after) action using the version history 
mechanism. This implementation of TRANSFER is more elaborate than the earlier one—it 
tests to see whether or not the account to be debited has enough funds to cover the trans­
fer and if not it aborts the action. The order of steps in the transfer procedure is 
remarkably unconstrained by any consideration other than calculating the correct 
answer. The reading of credit_account, for example, could casually be moved to any 
point between NEW_ACTION and the place where yvalue is recalculated. We conclude that 
the journal storage system has made the pre-commit discipline much less onerous than 
we might have expected. 
There is still one loose end: it is essential that updates to a version history and changes 
to an outcome record be all-or-nothing. That is, if the system fails while the thread is 
inside WRITE_NEW_VALUE, adjusting structures to append a new version, or inside COMMIT 
while updating the outcome record, the cell being written must not be muddled; it must 
either stay as it was before the crash or change to the intended new value. The solution 
is to design all modifications to the internal structures of journal storage so that they can 
Saltzer & Kaashoek Ch. 9, p. 36 
June 24, 2009 12:26 am

## Page 39

9.2 All-or-Nothing Atomicity I: Concepts 9–37  
be done by overwriting a single cell. For example, suppose that the name of a variable 
that has a version history refers to a cell that contains the address of the newest version, 
and that versions are linked from the newest version backwards, by address references. 
Adding a version consists of allocating space for a new version, reading the current 
address of the prior version, writing that address in the backward link field of the new 
version, and then updating the descriptor with the address of the new version. That last 
update can be done by overwriting a single cell. Similarly, updating an outcome record 
to change it from PENDING to COMMITTED can be done by overwriting a single cell. 
As a first bootstrapping step, we have reduced the general problem of creating all-or­
nothing actions to the specific problem of doing an all-or-nothing overwrite of one cell. 
As the remaining bootstrapping step, recall that we already know two ways to do a single-
cell all-or-nothing overwrite: apply the ALL_OR_NOTHING_PUT procedure of Figure 9.7. (If 
there is concurrency, updates to the internal structures of the version history also need 
before-or-after atomicity. Section 9.4 will explore methods of providing it.) 
9.2.4 How Version Histories are Used 
The careful reader will note two possibly puzzling things about the version history 
scheme just described. Both will become less puzzling when we discuss concurrency and 
before-or-after atomicity in Section 9.4 of this chapter: 
1.  Because READ_CURRENT_VALUE skips over any version belonging to another all-or­
nothing action whose OUTCOME record is not COMMITTED, it isn’t really necessary to 
change the OUTCOME record when an all-or-nothing action aborts; the record could 
just remain in the PENDING state indeﬁnitely. However, when we introduce 
concurrency, we will ﬁnd that a pending action may prevent other threads from 
reading variables for which the pending action created a new version, so it will 
become important to distinguish aborted actions from those that really are still 
pending. 
2.  As we have deﬁned READ_CURRENT_VALUE, versions older than the most recent 
committed version are inaccessible and they might just as well be discarded. 
Discarding could be accomplished either as an additional step in the journal 
storage manager, or as part of a separate garbage collection activity. Alternatively, 
those older versions may be useful as an historical record, known as an archive, 
with the addition of timestamps on commit records and procedures that can locate 
and return old values created at speciﬁed times in the past. For this reason, a 
version history system is sometimes called a temporal database or is said to provide 
time domain addressing. The banking industry abounds in requirements that make 
use of history information, such as reporting a consistent sum of balances in all 
bank accounts, paying interest on the ﬁfteenth on balances as of the ﬁrst of the 
month, or calculating the average balance last month. Another reason for not 
discarding old versions immediately will emerge when we discuss concurrency and 
Saltzer & Kaashoek Ch. 9, p. 37  
June 24, 2009 12:26 am

## Page 40

9–38 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
before-or-after atomicity: concurrent threads may, for correctness, need to read old 
versions even after new versions have been created and committed. 
Direct implementation of a version history raises concerns about performance: rather 
than simply reading a named storage cell, one must instead make at least one indirect 
reference through a descriptor that locates the storage cell containing the current version. 
If the cell storage device is on a magnetic disk, this extra reference is a potential bottle­
neck, though it can be alleviated with a cache. A bottleneck that is harder to alleviate 
occurs on updates. Whenever an application writes a new value, the journal storage layer 
must allocate space in unused cell storage, write the new version, and update the version 
history descriptor so that future readers can find the new version. Several disk writes are 
likely to be required. These extra disk writes may be hidden inside the journal storage 
layer and with added cleverness may be delayed until commit and batched, but they still 
have a cost. When storage access delays are the performance bottleneck, extra accesses 
slow things down. 
In consequence, version histories are used primarily in low-performance applications. 
One common example is found in revision management systems used to coordinate 
teams doing program development. A programmer “checks out” a group of files, makes 
changes, and then “checks in” the result. The check-out and check-in operations are all-
or-nothing and check-in makes each changed file the latest version in a complete history 
of that file, in case a problem is discovered later. (The check-in operation also verifies that 
no one else changed the files while they were checked out, which catches some, but not 
all, coordination errors.) A second example is that some interactive applications such as 
word processors or image editing systems provide a “deep undo” feature, which allows a 
user who decides that his or her recent editing is misguided to step backwards to reach 
an earlier, satisfactory state. A third example appears in file systems that automatically 
create a new version every time any application opens an existing file for writing; when 
the application closes the file, the file system tags a number suffix to the name of the pre­
vious version of the file and moves the original name to the new version. These interfaces 
employ version histories because users find them easy to understand and they provide all-
or-nothing atomicity in the face of both system failures and user mistakes. Most such 
applications also provide an archive that is useful for reference and that allows going back 
to a known good version. 
Applications requiring high performance are a different story. They, too, require all-
or-nothing atomicity, but they usually achieve it by applying a specialized technique 
called a log. Logs are our next topic. 
9.3 All-or-Nothing Atomicity II: Pragmatics 
Database management applications such as airline reservation systems or banking sys­
tems usually require high performance as well as all-or-nothing atomicity, so their 
designers use streamlined atomicity techniques. The foremost of these techniques 
sharply separates the reading and writing of data from the failure recovery mechanism. 
Saltzer & Kaashoek Ch. 9, p. 38 
June 24, 2009 12:26 am

## Page 41

9.3 All-or-Nothing Atomicity II: Pragmatics 9–39  
The idea is to minimize the number of storage accesses required for the most common 
activities (application reads and updates). The trade-off is that the number of storage 
accesses for rarely-performed activities (failure recovery, which one hopes is actually exer­
cised only occasionally, if at all) may not be minimal. The technique is called logging. 
Logging is also used for purposes other than atomicity, several of which Sidebar 9.4 
describes. 
9.3.1 Atomicity Logs 
The basic idea behind atomicity logging is to combine the all-or-nothing atomicity of 
journal storage with the speed of cell storage, by having the application twice record every 
change to data. The application first logs the change in journal storage, and then it installs 
the change in cell storage*. One might think that writing data twice must be more expen­
sive than writing it just once into a version history, but the separation permits specialized 
optimizations that can make the overall system faster. 
The first recording, to journal storage, is optimized for fast writing by creating a sin­
gle, interleaved version history of all variables, known as a log. The information 
describing each data update forms a record that the application appends to the end of the 
log. Since there is only one log, a single pointer to the end of the log is all that is needed 
to find the place to append the record of a change of any variable in the system. If the 
log medium is magnetic disk, and the disk is used only for logging, and the disk storage 
management system allocates sectors contiguously, the disk seek arm will need to move 
only when a disk cylinder is full, thus eliminating most seek delays. As we will see, recov­
ery does involve scanning the log, which is expensive, but recovery should be a rare event. 
Using a log is thus an example of following the hint to optimize for the common case. 
The second recording, to cell storage, is optimized to make reading fast: the applica­
tion installs by simply overwriting the previous cell storage record of that variable. The 
record kept in cell storage can be thought of as a cache that, for reading, bypasses the 
effort that would be otherwise be required to locate the latest version in the log. In addi­
tion, by not reading from the log the logging disk’s seek arm can remain in position, 
ready for the next update. The two steps, LOG and INSTALL, become a different implemen­
tation of the WRITE_NEW_VALUE interface of Figure 9.11. Figure 9.17 illustrates this two-
step implementation. 
The underlying idea is that the log is the authoritative record of the outcome of the 
action. Cell storage is merely a reference copy; if it is lost, it can be reconstructed from 
the log. The purpose of installing a copy in cell storage is to make both logging and read­
ing faster. By recording data twice, we obtain high performance in writing, high 
performance in reading, and all-or-nothing atomicity, all at the same time. 
There are three common logging configurations, shown in Figure 9.18. In each of 
these three configurations, the log resides in non-volatile storage. For the in-memory 
* A hardware architect would say “…it graduates the change to cell storage”. This text, somewhat 
arbitrarily, chooses to use the database management term “install” . 
Saltzer & Kaashoek Ch. 9, p. 39 
June 24, 2009 12:26 am

## Page 42

CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
9–40 
Sidebar 9.4:  The many uses of logs A log is an object whose primary usage method is to 
append a new record. Log implementations normally provide procedures to read entries from 
oldest to newest or in reverse order, but there is usually not any procedure for modifying 
previous entries. Logs are used for several quite distinct purposes, and this range of purposes 
sometimes gets confused in real-world designs and implementations. Here are some of the most 
common uses for logs: 
1. Atomicity log. If one logs the component actions of an all-or-nothing action, together with 
sufﬁcient before and after information, then a crash recovery procedure can undo (and thus 
roll back the effects of) all-or-nothing actions that didn’t get a chance to complete, or ﬁnish 
all-or-nothing actions that committed but that didn’t get a chance to record all of their 
effects. 
2. Archive log. If the log is kept indeﬁnitely, it becomes a place where old values of data and the 
sequence of actions taken by the system or its applications can be kept for review. There are 
many uses for archive information: watching for failure patterns, reviewing the actions of 
the system preceding and during a security breach, recovery from application-layer mistakes 
(e.g., a clerk incorrectly deleted an account), historical study, fraud control, and compliance 
with record-keeping requirements. 
3. Performance log. Most mechanical storage media have much higher performance for 
sequential access than for random access. Since logs are written sequentially, they are ideally 
suited to such storage media. It is possible to take advantage of this match to the physical 
properties of the media by structuring data to be written in the form of a log. When 
combined with a cache that eliminates most disk reads, a performance log can provide a 
signiﬁcant speed-up. As will be seen in the accompanying text, an atomicity log is usually 
also a performance log. 
4. Durability log. If the log is stored on a non-volatile medium—say magnetic tape—that fails 
in ways and at times that are independent from the failures of the cell storage medium— 
which might be magnetic disk—then the copies of data in the log are replicas that can be 
used as backup in case of damage to the copies of the data in cell storage. This kind of log 
helps implement durable storage. Any log that uses a non-volatile medium, whether 
intended for atomicity, archiving or performance, typically also helps support durability. 
It is essential to have these various purposes—all-or-nothing atomicity, archive, performance, 
and durable storage—distinct in one’s mind when examining or designing a log 
implementation because they lead to different priorities among design trade-offs. When 
archive is the goal, low cost of the storage medium is usually more important than quick access 
because archive logs are large but, in practice, infrequently read. When durable storage is the 
goal, it may be important to use storage media with different physical properties, so that failure 
modes will be as independent as possible. When all-or-nothing atomicity or performance is the 
purpose, minimizing mechanical movement of the storage device becomes a high priority. 
Because of the competing objectives of different kinds of logs, as a general rule, it is usually a 
wise move to implement separate, dedicated logs for different functions. 
Saltzer & Kaashoek Ch. 9, p. 40 
June 24, 2009 12:26 am

## Page 43

9.3 All-or-Nothing Atomicity II: Pragmatics 9–41  
Journal Storage 
WRITE_NEW_VALUE 
Log 
Cell 
log 
install 
Storage 
READ_CURRENT_VALUE 
current 
end of log 
FIGURE 9.17 
Logging for all-or-nothing atomicity. The application performs WRITE_NEW_VALUE by ﬁrst 
appending a record of the new value to the log in journal storage, and then installing the new 
value in cell storage by overwriting. The application performs READ_CURRENT_VALUE by reading 
just from cell storage. 
Volatile storage 
Non-volatile storage 
log
Application 
program 
cell 
storage 
In-memory database: 
log
Application 
program 
cell 
storage 
Ordinary database: 
Application 
program 
log 
cell 
storage 
cache
High-performance
database: 
FIGURE 9.18 
Three common logging conﬁgurations. Arrows show data ﬂow as the application reads, logs, 
and installs data. 
Saltzer & Kaashoek Ch. 9, p. 41 
June 24, 2009 12:26 am

## Page 44

9–42 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
database, cell storage resides entirely in some volatile storage medium. In the second 
common configuration, cell storage resides in non-volatile storage along with the log. 
Finally, high-performance database management systems usually blend the two preced­
ing configurations by implementing a cache for cell storage in a volatile medium, and a 
potentially independent multilevel memory management algorithm moves data between 
the cache and non-volatile cell storage. 
Recording everything twice adds one significant complication to all-or-nothing ato­
micity because the system can crash between the time a change is logged and the time it 
is installed. To maintain all-or-nothing atomicity, logging systems follow a protocol that 
has two fundamental requirements. The first requirement is a constraint on the order of 
logging and installing. The second requirement is to run an explicit recovery procedure 
after every crash. (We saw a preview of the strategy of using a recovery procedure in Fig­
ure 9.7, which used a recovery procedure named CHECK_AND_REPAIR.) 
9.3.2 Logging Protocols 
There are several kinds of atomicity logs that vary in the order in which things are done 
and in the details of information logged. However, all of them involve the ordering con­
straint implied by the numbering of the arrows in Figure 9.17. The constraint is a version 
of the golden rule of atomicity (never modify the only copy), known as the write-ahead­
log (WAL) protocol: 
Write-ahead-log protocol 
Log the update before installing it. 
The reason is that logging appends but installing overwrites. If an application violates 
this protocol by installing an update before logging it and then for some reason must 
abort, or the system crashes, there is no systematic way to discover the installed update 
and, if necessary, reverse it. The write-ahead-log protocol ensures that if a crash occurs, 
a recovery procedure can, by consulting the log, systematically find all completed and 
intended changes to cell storage and either restore those records to old values or set them 
to new values, as appropriate to the circumstance. 
The basic element of an atomicity log is the log record. Before an action that is to be 
all-or-nothing installs a data value, it appends to the end of the log a new record of type 
CHANGE containing, in the general case, three pieces of information (we will later see spe­
cial cases that allow omitting item 2 or item 3): 
1.  The identity of the all-or-nothing action that is performing the update. 
2.  A component action that, if performed, installs the intended value in cell storage. 
This component action is a kind of an insurance policy in case the system crashes. 
If the all-or-nothing action commits, but then the system crashes before the action 
has a chance to perform the install, the recovery procedure can perform the install 
Saltzer & Kaashoek Ch. 9, p. 42  
June 24, 2009 12:26 am

## Page 45

9.3 All-or-Nothing Atomicity II: Pragmatics 9–43  
on behalf of the action. Some systems call this component action the do action, 
others the redo action. For mnemonic compatibility with item 3, this text calls it 
the redo action. 
3.  A second component action that, if performed, reverses the effect on cell storage 
of the planned install. This component action is known as the undo action because 
if, after doing the install, the all-or-nothing action aborts or the system crashes, it 
may be necessary for the recovery procedure to reverse the effect of (undo) the 
install. 
An application appends a log record by invoking the lower-layer procedure LOG, which 
itself must be atomic. The LOG procedure is another example of bootstrapping: Starting 
with, for example, the ALL_OR_NOTHING_PUT described earlier in this chapter, a log designer 
creates a generic LOG procedure, and using the LOG procedure an application programmer 
then can implement all-or-nothing atomicity for any properly designed composite 
action. 
As we saw in Figure 9.17, LOG and INSTALL are the logging implementation of the 
WRITE_NEW_VALUE part of the interface of Figure 9.11, and READ_CURRENT_VALUE is simply a 
READ from cell storage. We also need a logging implementation of the remaining parts of 
the Figure 9.11 interface. The way to implement NEW_ACTION is to log a BEGIN record that 
contains just the new all-or-nothing action’s identity. As the all-or-nothing action pro­
ceeds through its pre-commit phase, it logs CHANGE records. To implement COMMIT or 
ABORT, the all-or-nothing action logs an OUTCOME record that becomes the authoritative 
indication of the outcome of the all-or-nothing action. The instant that the all-or-noth­
ing action logs the OUTCOME record is its commit point. As an example, Figure 9.19 shows 
our by now familiar TRANSFER action implemented with logging. 
Because the log is the authoritative record of the action, the all-or-nothing action can 
perform installs to cell storage at any convenient time that is consistent with the write-
ahead-log protocol, either before or after logging the OUTCOME record. The final step of an 
action is to log an END record, again containing just the action’s identity, to show that the 
action has completed all of its installs. (Logging all four kinds of activity—BEGIN, CHANGE, 
OUTCOME, and END—is more general than sometimes necessary. As we will see, some log­
ging systems can combine, e.g., OUTCOME and END, or BEGIN with the first CHANGE.) Figure 
9.20 shows examples of three log records, two typical CHANGE records of an all-or-nothing 
TRANSFER action, interleaved with the OUTCOME record of some other, perhaps completely 
unrelated, all-or-nothing action. 
One consequence of installing results in cell storage is that for an all-or-nothing 
action to abort it may have to do some clean-up work. Moreover, if the system involun­
tarily terminates a thread that is in the middle of an all-or-nothing action (because, for 
example, the thread has gotten into a deadlock or an endless loop) some entity other than 
the hapless thread must clean things up. If this clean-up step were omitted, the all-or­
nothing action could remain pending indefinitely. The system cannot simply ignore 
indefinitely pending actions because all-or-nothing actions initiated by other threads are 
likely to want to use the data that the terminated action changed. (This is actually a 
Saltzer & Kaashoek Ch. 9, p. 43  
June 24, 2009 12:26 am

## Page 46

9–44 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.19 
1 procedure TRANSFER (debit_account, credit_account, amount) 
2 
my_id ← LOG (BEGIN_TRANSACTION) 
3 
dbvalue.old ← GET (debit_account) 
4 
dbvalue.new ← dbvalue.old - amount 
5 
crvalue.old ← GET (credit_account, my_id) 
6 
crvalue.new ← crvalue.old + amount 
7 
LOG (CHANGE, my_id, 
8 
“PUT (debit_account, dbvalue.new)”, 
//redo action 
9 
“PUT (debit_account, dbvalue.old)” ) 
//undo action 
10 
LOG ( CHANGE, my_id, 
11 
“PUT (credit_account, crvalue.new)” 
//redo action 
12 
“PUT (credit_account, crvalue.old)”) 
//undo action 
13 
PUT (debit_account, dbvalue.new) 
// install 
14 
PUT (credit_account, crvalue.new) 
// install 
15 
if dbvalue.new > 0 then 
16 
LOG ( OUTCOME, COMMIT, my_id) 
17 
else 
18 
LOG (OUTCOME, ABORT, my_id) 
19 
signal(“Action not allowed. Would make debit account negative.”) 
20 
LOG (END_TRANSACTION, my_id) 
An all-or-nothing TRANSFER procedure, implemented with logging. 
before-or-after atomicity concern, one of the places where all-or-nothing atomicity and 
before-or-after atomicity intersect.) 
If the action being aborted did any installs, those installs are still in cell storage, so 
simply appending to the log an OUTCOME record saying that the action aborted is not 
enough to make it appear to later observers that the all-or-nothing action did nothing. 
The solution to this problem is to execute a generic ABORT procedure. The ABORT proce­
… 
9979 
PUT(debit_account, $120) 
action_id: 
redo_action: 
undo_action: 
CHANGE
type: 
PUT(debit_account, $90) 
9974 
COMMITTED 
action_id: 
status: 
OUTCOME
type: 
9979 
PUT(credit_account, $40) 
PUT(credit_account, $10) 
action_id: 
redo_action: 
undo_action: 
CHANGE
type: 
older log records 
newer log records 
FIGURE 9.20 
An example of a section of an atomicity log, showing two CHANGE records for a TRANSFER action 
that has action_id 9979 and the OUTCOME record of a different all-or-nothing action. 
Saltzer & Kaashoek Ch. 9, p. 44 
June 24, 2009 12:26 am

## Page 47

9.3 All-or-Nothing Atomicity II: Pragmatics 9–45  
dure restores to their old values all cell storage variables that the all-or-nothing action 
installed. The ABORT procedure simply scans the log backwards looking for log entries cre­
ated by this all-or-nothing action; for each CHANGE record it finds, it performs the logged 
undo_action, thus restoring the old values in cell storage. The backward search terminates 
when the ABORT procedure finds that all-or-nothing action’s BEGIN record. Figure 9.21 
illustrates. 
The extra work required to undo cell storage installs when an all-or-nothing action 
aborts is another example of optimizing for the common case: one expects that most all-or­
nothing actions will commit, and that aborted actions should be relatively rare. The extra 
effort of an occasional roll back of cell storage values will (one hopes) be more than repaid 
by the more frequent gains in performance on updates, reads, and commits. 
9.3.3 Recovery Procedures 
The write-ahead log protocol is the first of the two required protocol elements of a log­
ging system. The second required protocol element is that, following every system crash, 
the system must run a recovery procedure before it allows ordinary applications to use 
the data. The details of the recovery procedure depend on the particular configuration 
of the journal and cell storage with respect to volatile and non-volatile memory. 
Consider first recovery for the in-memory database of Figure 9.18. Since a system 
crash may corrupt anything that is in volatile memory, including both the state of cell 
storage and the state of any currently running threads, restarting a crashed system usually 
begins by resetting all volatile memory. The effect of this reset is to abandon both the cell 
1 procedure ABORT (action_id)  
2
 starting at end of log repeat until beginning  
3 
log_record ← previous record of log  
4 
if log_record.id = action_id then  
5 
if (log_record.type = OUTCOME)  
6 
then signal (“Can’t abort an already completed action.”)  
7 
if (log_record.type = CHANGE)  
8 
then perform undo_action of log_record  
9 
if (log_record.type = BEGIN)   
10 
then break repeat  
11 
LOG (action_id, OUTCOME, ABORTED)
 // Block future undos.  
12 
LOG (action_id, END)  
FIGURE 9.21 
Generic ABORT procedure for a logging system.The argument action_id identiﬁes the action to 
be aborted. An atomic action calls this procedure if it decides to abort. In addition, the operating 
system may call this procedure if it decides to terminate the action, for example to break a 
deadlock or because the action is running too long. The LOG procedure must itself be atomic. 
Saltzer & Kaashoek Ch. 9, p. 45 
June 24, 2009 12:26 am

## Page 48

9–46 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.22 
1 procedure RECOVER () // Recovery procedure for a volatile, in-memory database. 
2 
winners ← NULL 
3 
starting at end of log repeat until beginning 
4 
log_record ← previous record of log 
5 
if (log_record.type = OUTCOME) 
6 
then winners ← winners + log_record 
// Set addition. 
7 
starting at beginning of log repeat until end 
8 
log_record ← next record of log 
9 
if (log_record.type= CHANGE) 
10 
and (outcome_record ← find (log_record.action_id) in winners) 
11 
and (outcome_record.status = COMMITTED) then 
12 
perform log_record.redo_action 
An idempotent redo-only recovery procedure for an in-memory database. Because RECOVER 
writes only to volatile storage, if a crash occurs while it is running it is safe to run it again. 
storage version of the database and any all-or-nothing actions that were in progress at the 
time of the crash. On the other hand, the log, since it resides on non-volatile journal stor­
age, is unaffected by the crash and should still be intact. 
The simplest recovery procedure performs two passes through the log. On the first 
pass, it scans the log backward from the last record, so the first evidence it will encounter 
of each all-or-nothing action is the last record that the all-or-nothing action logged. A 
backward log scan is sometimes called a LIFO (for last-in, first-out) log review. As the 
recovery procedure scans backward, it collects in a set the identity and completion status 
of every all-or-nothing action that logged an OUTCOME record before the crash. These 
actions, whether committed or aborted, are known as winners. 
When the backward scan is complete the set of winners is also complete, and the 
recovery procedure begins a forward scan of the log. The reason the forward scan is 
needed is that restarting after the crash completely reset the cell storage. During the for­
ward scan the recovery procedure performs, in the order found in the log, all of the REDO 
actions of every winner whose OUTCOME record says that it COMMITTED. Those REDOs reinstall 
all committed values in cell storage, so at the end of this scan, the recovery procedure has 
restored cell storage to a desirable state. This state is as if every all-or-nothing action that 
committed before the crash had run to completion, while every all-or-nothing action 
that aborted or that was still pending at crash time had never existed. The database sys­
tem can now open for regular business. Figure 9.22 illustrates. 
This recovery procedure emphasizes the point that a log can be viewed as an author­
itative version of the entire database, sufficient to completely reconstruct the reference 
copy in cell storage. 
There exist cases for which this recovery procedure may be overkill, when the dura­
bility requirement of the data is minimal. For example, the all-or-nothing action may 
Saltzer & Kaashoek Ch. 9, p. 46 
June 24, 2009 12:26 am

## Page 49

9.3 All-or-Nothing Atomicity II: Pragmatics 9–47  
have been to make a group of changes to soft state in volatile storage. If the soft state is 
completely lost in a crash, there would be no need to redo installs because the definition 
of soft state is that the application is prepared to construct new soft state following a 
crash. Put another way, given the options of “all” or “nothing,” when the data is all soft 
state “nothing” is always an appropriate outcome after a crash. 
A critical design property of the recovery procedure is that, if there should be another 
system crash during recovery, it must still be possible to recover. Moreover, it must be 
possible for any number of crash-restart cycles to occur without compromising the cor­
rectness of the ultimate result. The method is to design the recovery procedure to be 
idempotent. That is, design it so that if it is interrupted and restarted from the beginning 
it will produce exactly the same result as if it had run to completion to begin with. With 
the in-memory database configuration, this goal is an easy one: just make sure that the 
recovery procedure modifies only volatile storage. Then, if a crash occurs during recov­
ery, the loss of volatile storage automatically restores the state of the system to the way it 
was when the recovery started, and it is safe to run it again from the beginning. If the 
recovery procedure ever finishes, the state of the cell storage copy of the database will be 
correct, no matter how many interruptions and restarts intervened. 
The ABORT procedure similarly needs to be idempotent because if an all-or-nothing 
action decides to abort and, while running ABORT, some timer expires, the system may 
decide to terminate and call ABORT for that same all-or-nothing action. The version of 
abort in Figure 9.21 will satisfy this requirement if the individual undo actions are them­
selves idempotent. 
9.3.4 Other Logging Conﬁgurations: Non-Volatile Cell Storage 
Placing cell storage in volatile memory is a sweeping simpliﬁcation that works well for 
small and medium-sized databases, but some databases are too large for that to be prac­
tical, so the designer finds it necessary to place cell storage on some cheaper, non-volatile 
storage medium such as magnetic disk, as in the second configuration of Figure 9.18. But 
with a non-volatile storage medium, installs survive system crashes, so the simple recov­
ery procedure used with the in-memory database would have two shortcomings: 
1.  If, at the time of the crash, there were some pending all-or-nothing actions that 
had installed changes, those changes will survive the system crash. The recovery 
procedure must reverse the effects of those changes, just as if those actions had 
aborted. 
2.  That recovery procedure reinstalls the entire database, even though in this case 
much of it is probably intact in non-volatile storage. If the database is large enough 
that it requires non-volatile storage to contain it, the cost of unnecessarily 
reinstalling it in its entirety at every recovery is likely to be unacceptable. 
In addition, reads and writes to non-volatile cell storage are likely to be slow, so it is 
nearly always the case that the designer installs a cache in volatile memory, along with a 
Saltzer & Kaashoek Ch. 9, p. 47  
June 24, 2009 12:26 am

## Page 50

9–48 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
multilevel memory manager, thus moving to the third configuration of Figure 9.18. But 
that addition introduces yet another shortcoming: 
3.  In a multilevel memory system, the order in which data is written from volatile 
levels to non-volatile levels is generally under control of a multilevel memory 
manager, which may, for example, be running a least-recently-used algorithm. As 
a result, at the instant of the crash some things that were thought to have been 
installed may not yet have migrated to the non-volatile memory. 
To postpone consideration of this shortcoming, let us for the moment assume that 
the multilevel memory manager implements a write-through cache. (Section 9.3.6, 
below, will return to the case where the cache is not write-through.) With a write-
through cache, we can be certain that everything that the application program has 
installed has been written to non-volatile storage. This assumption temporarily drops the 
third shortcoming out of our list of concerns and the situation is the same as if we were 
using the “Ordinary Database” configuration of Figure 9.18 with no cache. But we still 
have to do something about the first two shortcomings, and we also must make sure that 
the modified recovery procedure is still idempotent. 
To address the first shortcoming, that the database may contain installs from actions 
that should be undone, we need to modify the recovery procedure of Figure 9.22. As the 
recovery procedure performs its initial backward scan, rather than looking for winners, 
it instead collects in a set the identity of those all-or-nothing actions that were still in 
progress at the time of the crash. The actions in this set are known as losers, and they can 
include both actions that committed and actions that did not. Losers are easy to identify 
because the first log record that contains their identity that is encountered in a backward 
scan will be something other than an END record. To identify the losers, the pseudocode 
keeps track of which actions logged an END record in an auxiliary list named completeds. 
When RECOVER comes across a log record belong to an action that is not in completed, it 
adds that action to the set named losers. In addition, as it scans backwards, whenever the 
recovery procedure encounters a CHANGE record belonging to a loser, it performs the UNDO 
action listed in the record. In the course of the LIFO log review, all of the installs per­
formed by losers will thus be rolled back and the state of the cell storage will be as if the 
all-or-nothing actions of losers had never started. Next, RECOVER performs the forward log 
scan of the log, performing the redo actions of the all-or-nothing actions that committed, 
as shown in Figure 9.23. Finally, the recovery procedure logs an END record for every all-
or-nothing action in the list of losers. This END record transforms the loser into a com­
pleted action, thus ensuring that future recoveries will ignore it and not perform its 
undos again. For future recoveries to ignore aborted losers is not just a performance 
enhancement, it is essential, to avoid incorrectly undoing updates to those same variables 
made by future all-or-nothing actions. 
As before, the recovery procedure must be idempotent, so that if a crash occurs during 
recovery the system can just run the recovery procedure again. In addition to the tech­
nique used earlier of placing the temporary variables of the recovery procedure in volatile 
storage, each individual undo action must also be idempotent. For this reason, both redo 
Saltzer & Kaashoek Ch. 9, p. 48  
June 24, 2009 12:26 am

## Page 51

9.3 All-or-Nothing Atomicity II: Pragmatics 9–49  
1 procedure RECOVER ()// Recovery procedure for non-volatile cell memory 
2 
completeds ← NULL 
3 
losers ← NULL 
4 
starting at end of log repeat until beginning 
5 
log_record ← previous record of log 
6 
if (log_record.type = END) 
7 
then completeds ← completeds + log_record 
// Set addition. 
8 
if (log_record.action_id is not in completeds) then 
9 
losers ← losers + log_record 
// Add if not already in set. 
10 
if (log_record.type = CHANGE) then 
11 
perform log_record.undo_action 
12 
starting at beginning of log repeat until end 
13 
log_record ← next record of log 
14 
if (log_record.type = CHANGE) 
15 
and (log_record.action_id.status = COMMITTED) then 
16 
perform log_record.redo_action 
17 
for each log_record in losers do 
18 
log (log_record.action_id, END) 
// Show action completed. 
FIGURE 9.23 
An idempotent undo/redo recovery procedure for a system that performs installs to non-volatile 
cell memory. In this recovery procedure, losers are all-or-nothing actions that were in progress 
at the time of the crash. 
and undo actions are usually expressed as blind writes. A blind write is a simple overwrit­
ing of a data value without reference to its previous value. Because a blind write is 
inherently idempotent, no matter how many times one repeats it, the result is always the 
same. Thus, if a crash occurs part way through the logging of END records of losers, imme­
diately rerunning the recovery procedure will still leave the database correct. Any losers 
that now have END records will be treated as completed on the rerun, but that is OK 
because the previous attempt of the recovery procedure has already undone their installs. 
As for the second shortcoming, that the recovery procedure unnecessarily redoes 
every install, even installs not belong to losers, we can significantly simplify (and speed 
up) recovery by analyzing why we have to redo any installs at all. The reason is that, 
although the WAL protocol requires logging of changes to occur before install, there is 
no necessary ordering between commit and install. Until a committed action logs its END 
record, there is no assurance that any particular install of that action has actually hap­
pened yet. On the other hand, any committed action that has logged an END record has 
completed its installs. The conclusion is that the recovery procedure does not need to 
Saltzer & Kaashoek Ch. 9, p. 49 
June 24, 2009 12:26 am

## Page 52

9–50 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.24 
1 procedure RECOVER () 
// Recovery procedure for rollback recovery. 
2 
completeds ← NULL 
3 
losers ← NULL 
4 
starting at end of log repeat until beginning 
// Perform undo scan. 
5 
log_record ← previous record of log 
6 
if (log_record.type = OUTCOME) 
7 
then completeds ← completeds + log_record 
// Set addition. 
8 
if (log_record.action_id is not in completeds) then 
9 
losers ← losers + log_record 
// New loser. 
10 
if (log_record.type = CHANGE) then 
11 
perform log_record.undo_action 
12 
for each log_record in losers do 
13 
log (log_record.action_id, OUTCOME, ABORT) 
// Block future undos. 
An idempotent undo-only recovery procedure for rollback logging. 
redo installs for any committed action that has logged its END record. A useful exercise is 
to modify the procedure of Figure 9.23 to take advantage of that observation. 
It would be even better if the recovery procedure never had to redo any installs. We 
can arrange for that by placing another requirement on the application: it must perform 
all of its installs before it logs its OUTCOME record. That requirement, together with the 
write-through cache, ensures that the installs of every completed all-or-nothing action 
are safely in non-volatile cell storage and there is thus never a need to perform any redo 
actions. (It also means that there is no need to log an END record.) The result is that the 
recovery procedure needs only to undo the installs of losers, and it can skip the entire 
forward scan, leading to the simpler recovery procedure of Figure 9.24. This scheme, 
because it requires only undos, is sometimes called undo logging or rollback recovery. A 
property of rollback recovery is that for completed actions, cell storage is just as author­
itative as the log. As a result, one can garbage collect the log, discarding the log records 
of completed actions. The now much smaller log may then be able to fit in a faster stor­
age medium for which the durability requirement is only that it outlast pending actions. 
There is an alternative, symmetric constraint used by some logging systems. Rather 
than requiring that all installs be done before logging the OUTCOME record, one can instead 
require that all installs be done after recording the OUTCOME record. With this constraint, 
the set of CHANGE records in the log that belong to that all-or-nothing action become a 
description of its intentions. If there is a crash before logging an OUTCOME record, we know 
that no installs have happened, so the recovery never needs to perform any undos. On 
the other hand, it may have to perform installs for all-or-nothing actions that committed. 
This scheme is called redo logging or roll-forward recovery. Furthermore, because we are 
uncertain about which installs actually have taken place, the recovery procedure must 
Saltzer & Kaashoek Ch. 9, p. 50 
June 24, 2009 12:26 am

## Page 53

9.3 All-or-Nothing Atomicity II: Pragmatics 9–51  
perform all logged installs for all-or-nothing actions that did not log an END record. Any 
all-or-nothing action that logged an END record must have completed all of its installs, so 
there is no need for the recovery procedure to perform them. The recovery procedure 
thus reduces to doing installs just for all-or-nothing actions that were interrupted 
between the logging of their OUTCOME and END records. Recovery with redo logging can 
thus be quite swift, though it does require both a backward and forward scan of the entire 
log. 
We can summarize the procedures for atomicity logging as follows: 
•   Log to journal storage before installing in cell storage (WAL protocol) 
•  If all-or-nothing actions perform  all installs to non-volatile storage before 
logging their OUTCOME record, then recovery needs only to undo the installs of 
incomplete uncommitted actions. (rollback/undo recovery) 
•   If all-or-nothing actions perform no installs to non-volatile storage before logging 
their OUTCOME record, then recovery needs only to redo the installs of incomplete 
committed actions. (roll-forward/redo recovery) 
•   If all-or-nothing actions are not disciplined about when they do installs to non­
volatile storage, then recovery needs to both redo the installs of incomplete 
committed actions and undo the installs of incomplete uncommitted ones. 
In addition to reading and updating memory, an all-or-nothing action may also need 
to send messages, for example, to report its success to the outside world. The action of 
sending a message is just like any other component action of the all-or-nothing action. 
To provide all-or-nothing atomicity, message sending can be handled in a way analogous 
to memory update. That is, log a CHANGE record with a redo action that sends the message. 
If a crash occurs after the all-or-nothing action commits, the recovery procedure will per­
form this redo action along with other redo actions that perform installs. In principle, 
one could also log an undo_action that sends a compensating message (“Please ignore my 
previous communication!”). However, an all-or-nothing action will usually be careful 
not to actually send any messages until after the action commits, so roll-forward recovery 
applies. For this reason, a designer would not normally specify an undo action for a mes­
sage or for any other action that has outside-world visibility such as printing a receipt, 
opening a cash drawer, drilling a hole, or firing a missile. 
Incidentally, although much of the professional literature about database atomicity 
and recovery uses the terms “winner” and “loser” to describe the recovery procedure, dif­
ferent recovery systems use subtly different definitions for the two sets, depending on the 
exact logging scheme, so it is a good idea to review those definitions carefully. 
9.3.5 Checkpoints 
Constraining the order of installs to be all before or all after the logging of the OUTCOME 
record is not the only thing we could do to speed up recovery. Another technique that 
can shorten the log scan is to occasionally write some additional information, known as 
a checkpoint, to non-volatile storage. Although the principle is always the same, the exact 
Saltzer & Kaashoek Ch. 9, p. 51  
June 24, 2009 12:26 am

## Page 54

9–52 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
information that is placed in a checkpoint varies from one system to another. A check­
point can include information written either to cell storage or to the log (where it is 
known as a checkpoint record) or both. 
Suppose, for example, that the logging system maintains in volatile memory a list of 
identifiers of all-or-nothing actions that have started but have not yet recorded an END 
record, together with their pending/committed/aborted status, keeping it up to date by 
observing logging calls. The logging system then occasionally logs this list as a CHECKPOINT 
record. When a crash occurs sometime later, the recovery procedure begins a LIFO log 
scan as usual, collecting the sets of completed actions and losers. When it comes to a 
CHECKPOINT record it can immediately fill out the set of losers by adding those all-or-noth­
ing actions that were listed in the checkpoint that did not later log an END record. This 
list may include some all-or-nothing actions listed in the CHECKPOINT record as COMMITTED, 
but that did not log an END record by the time of the crash. Their installs still need to be 
performed, so they need to be added to the set of losers. The LIFO scan continues, but 
only until it has found the BEGIN record of every loser. 
With the addition of CHECKPOINT records, the recovery procedure becomes more com­
plex, but is potentially shorter in time and effort: 
1.  Do a LIFO scan of the log back to the last CHECKPOINT record, collecting identiﬁers 
of losers and undoing all actions they logged. 
2.  Complete the list of losers from information in the checkpoint. 
3.  Continue the LIFO scan, undoing the actions of losers, until every BEGIN record 
belonging to every loser has been found. 
4.  Perform a forward scan from that point to the end of the log, performing any 
committed actions belonging to all-or-nothing actions in the list of losers that 
logged an OUTCOME record with status COMMITTED. 
In systems in which long-running all-or-nothing actions are uncommon, step 3 will typ­
ically be quite brief or even empty, greatly shortening recovery. A good exercise is to 
modify the recovery program of Figure 9.23 to accommodate checkpoints. 
Checkpoints are also used with in-memory databases, to provide durability without 
the need to reprocess the entire log after every system crash. A useful checkpoint proce­
dure for an in-memory database is to make a snapshot of the complete database, writing 
it to one of two alternating (for all-or-nothing atomicity) dedicated non-volatile storage 
regions, and then logging a CHECKPOINT record that contains the address of the latest snap­
shot. Recovery then involves scanning the log back to the most recent CHECKPOINT record, 
collecting a list of committed all-or-nothing actions, restoring the snapshot described 
there, and then performing redo actions of those committed actions from the CHECKPOINT 
record to the end of the log. The main challenge in this scenario is dealing with update 
activity that is concurrent with the writing of the snapshot. That challenge can be met 
either by preventing all updates for the duration of the snapshot or by applying more 
complex before-or-after atomicity techniques such as those described in later sections of 
this chapter. 
Saltzer & Kaashoek Ch. 9, p. 52  
June 24, 2009 12:26 am

## Page 55

9.3 All-or-Nothing Atomicity II: Pragmatics 9–53  
9.3.6 What if the Cache is not Write-Through? (Advanced Topic) 
Between the log and the write-through cache, the logging configurations just described 
require, for every data update, two synchronous writes to non-volatile storage, with 
attendant delays waiting for the writes to complete. Since the original reason for intro­
ducing a log was to increase performance, these two synchronous write delays usually 
become the system performance bottleneck. Designers who are interested in maximizing 
performance would prefer to use a cache that is not write-through, so that writes can be 
deferred until a convenient time when they can be done in batches. Unfortunately, the 
application then loses control of the order in which things are actually written to non­
volatile storage. Loss of control of order has a significant impact on our all-or-nothing 
atomicity algorithms, since they require, for correctness, constraints on the order of 
writes and certainty about which writes have been done. 
The first concern is for the log itself because the write-ahead log protocol requires that 
appending a CHANGE record to the log precede the corresponding install in cell storage. 
One simple way to enforce the WAL protocol is to make just log writes write-through, 
but allow cell storage writes to occur whenever the cache manager finds it convenient. 
However, this relaxation means that if the system crashes there is no assurance that any 
particular install has actually migrated to non-volatile storage. The recovery procedure, 
assuming the worst, cannot take advantage of checkpoints and must again perform 
installs starting from the beginning of the log. To avoid that possibility, the usual design 
response is to flush the cache as part of logging each checkpoint record. Unfortunately, 
flushing the cache and logging the checkpoint must be done as a before-or-after action 
to avoid getting tangled with concurrent updates, which creates another design chal­
lenge. This challenge is surmountable, but the complexity is increasing. 
Some systems pursue performance even farther. A popular technique is to write the 
log to a volatile buffer, and force that entire buffer to non-volatile storage only when an 
all-or-nothing action commits. This strategy allows batching several CHANGE records with 
the next OUTCOME record in a single synchronous write. Although this step would appear 
to violate the write-ahead log protocol, that protocol can be restored by making the cache 
used for cell storage a bit more elaborate; its management algorithm must avoid writing 
back any install for which the corresponding log record is still in the volatile buffer. The 
trick is to number each log record in sequence, and tag each record in the cell storage 
cache with the sequence number of its log record. Whenever the system forces the log, it 
tells the cache manager the sequence number of the last log record that it wrote, and the 
cache manager is careful never to write back any cache record that is tagged with a higher 
log sequence number. 
We have in this section seen some good examples of the law of diminishing returns at 
work: schemes that improve performance sometimes require significantly increased com­
plexity. Before undertaking any such scheme, it is essential to evaluate carefully how 
much extra performance one stands to gain. 
Saltzer & Kaashoek Ch. 9, p. 53 
June 24, 2009 12:26 am

## Page 56

9–54 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
9.4 Before-or-After Atomicity I: Concepts 
The mechanisms developed in the previous sections of this chapter provide atomicity in 
the face of failure, so that other atomic actions that take place after the failure and sub­
sequent recovery find that an interrupted atomic action apparently either executed all of 
its steps or none of them. This and the next section investigate how to also provide ato­
micity of concurrent actions, known as before-or-after atomicity. In this development we 
will provide both all-or-nothing atomicity and before-or-after atomicity, so we will now 
be able to call the resulting atomic actions transactions. 
Concurrency atomicity requires additional mechanism because when an atomic 
action installs data in cell storage, that data is immediately visible to all concurrent 
actions. Even though the version history mechanism can hide pending changes from 
concurrent atomic actions, they can read other variables that the first atomic action plans 
to change. Thus, the composite nature of a multiple-step atomic action may still be dis­
covered by a concurrent atomic action that happens to look at the value of a variable in 
the midst of execution of the first atomic action. Thus, making a composite action 
atomic with respect to concurrent threads—that is, making it a before-or-after action— 
requires further effort. 
Recall that Section 9.1.5 defined the operation of concurrent actions to be correct if 
every result is guaranteed to be one that could have been obtained by some purely serial appli­
cation of those same actions. So we are looking for techniques that guarantee to produce 
the same result as if concurrent actions had been applied serially, yet maximize the per­
formance that can be achieved by allowing concurrency. 
In this Section 9.4 we explore three successively better before-or-after atomicity 
schemes, where “better” means that the scheme allows more concurrency. To illustrate 
the concepts we return to version histories, which allow a straightforward and compel­
ling correctness argument for each scheme. Because version histories are rarely used in 
practice, in the following Section 9.5 we examine a somewhat different approach, locks, 
which are widely used because they can provide higher performance, but for which cor­
rectness arguments are more difficult. 
9.4.1 Achieving Before-or-After Atomicity: Simple Serialization 
A version history assigns a unique identifier to each atomic action so that it can link ten­
tative versions of variables to the action’s outcome record. Suppose that we require that 
the unique identifiers be consecutive integers, which we interpret as serial numbers, and 
we modify the procedure BEGIN_TRANSACTION by adding enforcement of the following sim­
ple serialization rule: each newly created transaction n must, before reading or writing any 
data, wait until the preceding transaction n – 1 has either committed or aborted. (To 
ensure that there is always a transaction n – 1, assume that the system was initialized by 
creating a transaction number zero with an OUTCOME record in the committed state.) Fig­
ure 9.25 shows this version of BEGIN_TRANSACTION. The scheme forces all transactions to 
execute in the serial order that threads happen to invoke BEGIN_TRANSACTION. Since that 
Saltzer & Kaashoek Ch. 9, p. 54 
June 24, 2009 12:26 am

## Page 57

9.4 Before-or-After Atomicity I: Concepts 9–55  
1 procedure BEGIN_TRANSACTION () 
2
 id
 
← NEW_OUTCOME_RECORD (PENDING) 
// Create, initialize, assign id. 
3 
previous_id ← id – 1 
4 
wait until previous_id.outcome_record.state ≠ PENDING 
5 
return id 
FIGURE 9.25 
BEGIN_TRANSACTION with the simple serialization discipline to achieve before-or-after atomicity. 
In order that there be an id – 1 for every value of id, startup of the system must include creating 
a dummy transaction with id = 0 and id.outcome_record.state set to COMMITTED. Pseudocode 
for the procedure NEW_OUTCOME_RECORD appears in Figure 9.30. 
order is a possible serial order of the various transactions, by definition simple serializa­
tion will produce transactions that are serialized and thus are correct before-or-after 
actions. Simple serialization trivially provides before-or-after atomicity, and the transac­
tion is still all-or-nothing, so the transaction is now atomic both in the case of failure and 
in the presence of concurrency. 
Simple serialization provides before-or-after atomicity by being too conservative: it 
prevents all concurrency among transactions, even if they would not interfere with one 
another. Nevertheless, this approach actually has some practical value—in some applica­
tions it may be just the right thing to do, on the basis of simplicity. Concurrent threads 
can do much of their work in parallel because simple serialization comes into play only 
during those times that threads are executing transactions, which they generally would 
be only at the moments they are working with shared variables. If such moments are 
infrequent or if the actions that need before-or-after atomicity all modify the same small 
set of shared variables, simple serialization is likely to be just about as effective as any 
other scheme. In addition, by looking carefully at why it works, we can discover less con­
servative approaches that allow more concurrency, yet still have compelling arguments 
that they preserve correctness. Put another way, the remainder of study of before-or-after 
atomicity techniques is fundamentally nothing but invention and analysis of increasingly 
effective—and increasingly complex—performance improvement measures. 
The version history provides a useful representation for this analysis. Figure 9.26 
illustrates in a single figure the version histories of a banking system consisting of four 
accounts named A, B, C, and D, during the execution of six transactions, with serial num­
bers 1 through 6. The first transaction initializes all the objects to contain the value 0 and 
the following transactions transfer various amounts back and forth between pairs of 
accounts. 
This figure provides a straightforward interpretation of why simple serialization 
works correctly. Consider transaction 3, which must read and write objects B and C in 
order to transfer funds from one to the other. The way for transaction 3 to produce 
results as if it ran after transaction 2 is for all of 3’s input objects to have values that 
include all the effects of transaction 2—if transaction 2 commits, then any objects it 
Saltzer & Kaashoek Ch. 9, p. 55 
June 24, 2009 12:26 am

## Page 58

9–56 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
Object 
value of object at end of transaction 
A 
1 
0 
2 
+10 
3 
4 
+12 
5 
6 
0 
B 
0 
-10 
-6 
-12 
-2 
C 
0 
-4 
+2 
D 
0 
-2 
outcome 
record 
state 
Committed Committed Committed Aborted Committed Pending 
transaction 1: initialize all accounts to 0 
2: transfer 10 from B to A 
3: transfer 4 from C to B 
4: transfer 2 from D to A (aborts) 
5: transfer 6 from B to C 
6: transfer 10 from A to B 
FIGURE 9.26 
Version history of a banking system. 
changed and that 3 uses should have new values; if transaction 2 aborts, then any objects 
it tentatively changed and 3 uses should contain the values that they had when transac­
tion 2 started. Since in this example transaction 3 reads B and transaction 2 creates a new 
version of B, it is clear that for transaction 3 to produce a correct result it must wait until 
transaction 2 either commits or aborts. Simple serialization requires that wait, and thus 
ensures correctness. 
Figure 9.26 also provides some clues about how to increase concurrency. Looking at 
transaction 4 (the example shows that transaction 4 will ultimately abort for some reason, 
but suppose we are just starting transaction 4 and don’t know that yet), it is apparent that 
simple serialization is too strict. Transaction 4 reads values only from A and D, yet trans­
action 3 has no interest in either object. Thus the values of A and D will be the same 
whether or not transaction 3 commits, and a discipline that forces 4 to wait for 3’s com­
pletion delays 4 unnecessarily. On the other hand, transaction 4 does use an object that 
transaction 2 modifies, so transaction 4 must wait for transaction 2 to complete. Of 
course, simple serialization guarantees that, since transaction 4 can’t begin till transaction 
3 completes and transaction 3 couldn’t have started until transaction 2 completed. 
Saltzer & Kaashoek Ch. 9, p. 56 
June 24, 2009 12:26 am

## Page 59

9.4 Before-or-After Atomicity I: Concepts 9–57  
Object 
Value of object at end of transaction 
1 
2 
3
4
5
6 
7 
A 
B 
C 
D 
0 
0 
0 
0 
+10 
-10 
-6 
-4 
+12 
-2 
-12 
+2 
0 
-2 
+10 
+12 
0 
-6 
-2 
0 
-4 
+2 
+2 
0 
0 
-2 
-2 
-2 
OUTCOME 
record 
Committed Committed Committed Aborted Committed Pending 
Pending 
state 
Unchanged value 
Changed value 
FIGURE 9.27 
System state history with unchanged values shown. 
These observations suggest that there may be other, more relaxed, disciplines that can 
still guarantee correct results. They also suggest that any such discipline will probably 
involve detailed examination of exactly which objects each transaction reads and writes. 
Figure 9.26 represents the state history of the entire system in serialization order, but 
the slightly different representation of Figure 9.27 makes that state history more explicit. 
In Figure 9.27 it appears that each transaction has perversely created a new version of 
every object, with unchanged values in dotted boxes for those objects it did not actually 
change. This representation emphasizes that the vertical slot for, say, transaction 3 is in 
effect a reservation in the state history for every object in the system; transaction 3 has an 
opportunity to propose a new value for any object, if it so wishes. 
The reason that the system state history is helpful to the discussion is that as long as 
we eventually end up with a state history that has the values in the boxes as shown, the 
actual order in real time in which individual object values are placed in those boxes is 
unimportant. For example, in Figure 9.27, transaction 3 could create its new version of 
object C before transaction 2 creates its new version of B. We don’t care when things hap­
pen, as long as the result is to fill in the history with the same set of values that would 
result from strictly following this serial ordering. Making the actual time sequence unim-
Saltzer & Kaashoek Ch. 9, p. 57 
June 24, 2009 12:26 am

## Page 60

9–58 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
portant is exactly our goal, since that allows us to put concurrent threads to work on the 
various transactions. There are, of course, constraints on time ordering, but they become 
evident by examining the state history. 
Figure 9.27 allows us to see just what time constraints must be observed in order for 
the system state history to record this particular sequence of transactions. In order for a 
transaction to generate results appropriate for its position in the sequence, it should use 
as its input values the latest versions of all of its inputs. If Figure 9.27 were available, 
transaction 4 could scan back along the histories of its inputs A and D, to the most recent 
solid boxes (the ones created by transactions 2 and 1, respectively) and correctly conclude 
that if transactions 2 and 1 have committed then transaction 4 can proceed—even if 
transaction 3 hasn’t gotten around to filling in values for B and C and hasn’t decided 
whether or not it should commit. 
This observation suggests that any transaction has enough information to ensure 
before-or-after atomicity with respect to other transactions if it can discover the dotted-
versus-solid status of those version history boxes to its left. The observation also leads to 
a specific before-or-after atomicity discipline that will ensure correctness. We call this 
discipline mark-point. 
9.4.2 The Mark-Point Discipline 
Concurrent threads that invoke READ_CURRENT_VALUE as implemented in Figure 9.15 can 
not see a pending version of any variable. That observation is useful in designing a 
before-or-after atomicity discipline because it allows a transaction to reveal all of its 
results at once simply by changing the value of its OUTCOME record to COMMITTED. But in 
addition to that we need a way for later transactions that need to read a pending version 
to wait for it to become committed. The way to do that is to modify READ_CURRENT_VALUE 
to wait for, rather than skip over, pending versions created by transactions that are earlier 
in the sequential ordering (that is, they have a smaller caller_id), as implemented in lines 
4–9 of Figure 9.28. Because, with concurrency, a transaction later in the ordering may 
create a new version of the same variable before this transaction reads it, 
READ_CURRENT_VALUE still skips over any versions created by transactions that have a larger 
caller_id. Also, as before, it may be convenient to have a READ_MY_VALUE procedure (not 
shown) that returns pending values previously written by the running transaction. 
Adding the ability to wait for pending versions in READ_CURRENT_VALUE is the first step; 
to ensure correct before-or-after atomicity we also need to arrange that all variables that 
a transaction needs as inputs, but that earlier, not-yet-committed transactions plan to 
modify, have pending versions. To do that we call on the application programmer (for 
example, the programmer of the TRANSFER transaction) do a bit of extra work: each trans­
action should create new, pending versions of every variable it intends to modify, and 
announce when it is finished doing so. Creating a pending version has the effect of mark­
ing those variables that are not ready for reading by later transactions, so we will call the 
point at which a transaction has created them all the mark point of the transaction. The 
Saltzer & Kaashoek Ch. 9, p. 58 
June 24, 2009 12:26 am

## Page 61

9.4 Before-or-After Atomicity I: Concepts 9–59  
transaction announces that it has passed its mark point by calling a procedure named 
MARK_POINT_ANNOUNCE, which simply sets a flag in the outcome record for that transaction. 
The mark-point discipline then is that no transaction can begin reading its inputs 
until the preceding transaction has reached its mark point or is no longer pending. This 
discipline requires that each transaction identify which data it will update. If the trans­
action has to modify some data objects before it can discover the identity of others that 
require update, it could either delay setting its mark point until it does know all of the 
objects it will write (which would, of course, also delay all succeeding transactions) or use 
the more complex discipline described in the next section. 
For example, in Figure 9.27, the boxes under newly arrived transaction 7 are all dot­
ted; transaction 7 should begin by marking the ones that it plans to make solid. For 
convenience in marking, we split the WRITE_NEW_VALUE procedure of Figure 9.15 into two 
parts, named NEW_VERSION and WRITE_VALUE, as in Figure 9.29. Marking then consists sim­
ply of a series of calls to NEW_VERSION. When finished marking, the transaction calls 
MARK_POINT_ANNOUNCE. It may then go about its business, reading and writing values as 
appropriate to its purpose. 
Finally, we enforce the mark point discipline by putting a test and, depending on its 
outcome, a wait in BEGIN_TRANSACTION, as in Figure 9.30, so that no transaction may begin 
execution until the preceding transaction either reports that it has reached its mark point 
or is no longer 
PENDING. Figure 9.30 also illustrates an implementation of 
MARK_POINT_ANNOUNCE. No changes are needed in procedures ABORT and COMMIT as shown 
in Figure 9.13, so they are not repeated here. 
Because no transaction can start until the previous transaction reaches its mark point, 
all transactions earlier in the serial ordering must also have passed their mark points, so 
every transaction earlier in the serial ordering has already created all of the versions that 
it ever will. Since READ_CURRENT_VALUE now waits for earlier, pending values to become 
1 procedure READ_CURRENT_VALUE (data_id, this_transaction_id) 
2 
starting at end of data_id repeat until beginning 
3
v ← previous version of data_id 
4 
last_modifier ← v.action_id 
5 
if last_modifier ≥ this_transaction_id then skip v 
// Keep searching 
6 
wait until (last_modifier.outcome_record.state ≠ PENDING) 
7 
if (last_modifier.outcome_record.state = COMMITTED) 
8 
then return v.state 
9 
else skip v 
// Resume search 
10 
signal (“Tried to read an uninitialized variable”) 
FIGURE 9.28 
READ_CURRENT_VALUE for the mark-point discipline.This form of the procedure skips all versions 
created by transactions later than the calling transaction, and it waits for a pending version cre­
ated by an earlier transaction until that earlier transaction commits or aborts. 
Saltzer & Kaashoek Ch. 9, p. 59 
June 24, 2009 12:26 am

## Page 62

9–60 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.29 
1 procedure NEW_VERSION (reference data_id, this_transaction_id) 
2 
if this_transaction_id.outcome_record.mark_state = MARKED then 
3 
signal (“Tried to create new version after announcing mark point!”) 
4 
append new version v to data_id 
5 
v.value ← NULL 
6 
v.action_id ← transaction_id 
7 procedure WRITE_VALUE (reference data_id, new_value, this_transaction_id) 
8 
starting at end of data_id repeat until beginning 
9 
v ← previous version of data_id 
10 
if v.action_id = this_transaction_id 
11 
v.value ← new_value 
12 
return 
13 
signal (“Tried to write without creating new version!”)) 
Mark-point discipline versions of NEW_VERSION and WRITE_VALUE. 
FIGURE 9.30 
1 procedure BEGIN_TRANSACTION () 
2
 id
 
← NEW_OUTCOME_RECORD (PENDING) 
3 
previous_id ← id - 1 
4 
wait until (previous_id.outcome_record.mark_state = MARKED) 
5 
or (previous_id.outcome_record.state ≠ PENDING) 
6 
return id 
7 procedure NEW_OUTCOME_RECORD (starting_state) 
8 
ACQUIRE (outcome_record_lock) 
// Make this a before-or-after action. 
9
 id
 
← TICKET (outcome_record_sequencer) 
10 
allocate id.outcome_record 
11 
id.outcome_record.state ← starting_state 
12 
id.outcome_record.mark_state ← NULL 
13 
RELEASE (outcome_record_lock) 
14 
return id 
15 procedure MARK_POINT_ANNOUNCE (reference this_transaction_id) 
16 
this_transaction_id.outcome_record.mark_state ← MARKED 
The procedures BEGIN_TRANSACTION, NEW_OUTCOME_RECORD, and MARK_POINT_ANNOUNCE for the 
mark-point discipline. BEGIN_TRANSACTION presumes that there is always a preceding transac­
tion. so the system should be initialized by calling NEW_OUTCOME_RECORD to create an empty 
initial transaction in the starting_state COMMITTED and immediately calling 
MARK_POINT_ANNOUNCE for the empty transaction. 
Saltzer & Kaashoek Ch. 9, p. 60 
June 24, 2009 12:26 am

## Page 63

9.4 Before-or-After Atomicity I: Concepts 9–61  
committed or aborted, it will always return to its client a value that represents the final 
outcome of all preceding transactions. All input values to a transaction thus contain the 
committed result of all transactions that appear earlier in the serial ordering, just as if it 
had followed the simple serialization discipline. The result is thus guaranteed to be 
exactly the same as one produced by a serial ordering, no matter in what real time order 
the various transactions actually write data values into their version slots. The particular 
serial ordering that results from this discipline is, as in the case of the simple serialization 
discipline, the ordering in which the transactions were assigned serial numbers by 
NEW_OUTCOME_RECORD. 
There is one potential interaction between all-or-nothing atomicity and before-or­
after atomicity. If pending versions survive system crashes, at restart the system must 
track down all PENDING transaction records and mark them ABORTED to ensure that future 
invokers of READ_CURRENT_VALUE do not wait for the completion of transactions that have 
forever disappeared. 
The mark-point discipline provides before-or-after atomicity by bootstrapping from 
a more primitive before-or-after atomicity mechanism. As usual in bootstrapping, the 
idea is to reduce some general problem—here, that problem is to provide before-or-after 
atomicitiy for arbitrary application programs—to a special case that is amenable to a spe­
cial-case solution—here, the special case is construction and initialization of a new 
outcome record. The procedure NEW_OUTCOME_RECORD in Figure 9.30 must itself be a 
before-or-after action because it may be invoked concurrently by several different threads 
and it must be careful to give out different serial numbers to each of them. It must also 
create completely initialized outcome records, with value and mark_state set to PENDING 
and NULL, respectively, because a concurrent thread may immediately need to look at one 
of those fields. To achieve before-or-after atomicity, NEW_OUTCOME_RECORD bootstraps 
from the TICKET procedure of Section 5.6.3 to obtain the next sequential serial number, 
and it uses ACQUIRE and RELEASE to make its initialization steps a before-or-after action. 
Those procedures in turn bootstrap from still lower-level before-or-after atomicity mech­
anisms, so we have three layers of bootstrapping. 
We can now reprogram the funds TRANSFER procedure of Figure 9.15 to be atomic 
under both failure and concurrent activity, as in Figure 9.31. The major change from the 
earlier version is addition of lines 4 through 6, in which TRANSFER calls NEW_VERSION to 
mark the two variables that it intends to modify and then calls MARK_POINT_ANNOUNCE. The 
interesting observation about this program is that most of the work of making actions 
before-or-after is actually carried out in the called procedures. The only effort or thought 
required of the application programmer is to identify and mark, by creating new ver­
sions, the variables that the transaction will modify. 
The delays (which under the simple serialization discipline would all be concentrated 
in BEGIN_TRANSACTION) are distributed under the mark-point discipline. Some delays may 
still occur in BEGIN_TRANSACTION, waiting for the preceding transaction to reach its mark 
point. But if marking is done before any other calculations, transactions are likely to 
reach their mark points promptly, and thus this delay should be not as great as waiting 
for them to commit or abort. Delays can also occur at any invocation of 
Saltzer & Kaashoek Ch. 9, p. 61 
June 24, 2009 12:26 am

## Page 64

9–62 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.31 
1 procedure TRANSFER (reference debit_account, reference credit_account, 
2 
amount) 
3 
my_id ← BEGIN_TRANSACTION () 
4 
NEW_VERSION (debit_account, my_id) 
5 
NEW_VERSION (credit_account, my_id) 
6 
MARK_POINT_ANNOUNCE (my_id); 
7 
xvalue ← READ_CURRENT_VALUE (debit_account, my_id) 
8 
xvalue ← xvalue - amount 
9 
WRITE_VALUE (debit_account, xvalue, my_id) 
10 
yvalue ← READ_CURRENT_VALUE (credit_account, my_id) 
11 
yvalue ← yvalue + amount 
12 
WRITE_VALUE (credit_account, yvalue, my_id) 
13 
if xvalue > 0 then 
14 
COMMIT (my_id) 
15 
else 
16 
ABORT (my_id) 
17 
signal(“Negative transfers are not allowed.”) 
An implementation of the funds transfer procedure that uses the mark point discipline to ensure 
that it is atomic both with respect to failure and with respect to concurrent activity. 
READ_CURRENT_VALUE, but only if there is really something that the transaction must wait 
for, such as committing a pending version of a necessary input variable. Thus the overall 
delay for any given transaction should never be more than that imposed by the simple 
serialization discipline, and one might anticipate that it will often be less. 
A useful property of the mark-point discipline is that it never creates deadlocks. 
Whenever a wait occurs it is a wait for some transaction earlier in the serialization. That 
transaction may in turn be waiting for a still earlier transaction, but since no one ever 
waits for a transaction later in the ordering, progress is guaranteed. The reason is that at 
all times there must be some earliest pending transaction. The ordering property guar­
antees that this earliest pending transaction will encounter no waits for other transactions 
to complete, so it, at least, can make progress. When it completes, some other transaction 
in the ordering becomes earliest, and it now can make progress. Eventually, by this argu­
ment, every transaction will be able to make progress. This kind of reasoning about 
progress is a helpful element of a before-or-after atomicity discipline. In Section 9.5 of 
this chapter we will encounter before-or-after atomicity disciplines that are correct in the 
sense that they guarantee the same result as a serial ordering, but they do not guarantee 
progress. Such disciplines require additional mechanisms to ensure that threads do not 
end up deadlocked, waiting for one another forever. 
Two other minor points are worth noting. First, if transactions wait to announce 
their mark point until they are ready to commit or abort, the mark-point discipline 
reduces to the simple serialization discipline. That observation confirms that one disci-
Saltzer & Kaashoek Ch. 9, p. 62 
June 24, 2009 12:26 am

## Page 65

9.4 Before-or-After Atomicity I: Concepts 9–63  
pline is a relaxed version of the other. Second, there are at least two opportunities in the 
mark-point discipline to discover and report protocol errors to clients. A transaction 
should never call NEW_VERSION after announcing its mark point. Similarly, WRITE_VALUE 
can report an error if the client tries to write a value for which a new version was never 
created. Both of these error-reporting opportunities are implemented in the pseudocode 
of Figure 9.29. 
9.4.3 Optimistic Atomicity: Read-Capture (Advanced Topic) 
Both the simple serialization and mark-point disciplines are concurrency control meth­
ods that may be described as pessimistic. That means that they presume that interference 
between concurrent transactions is likely and they actively prevent any possibility of 
interference by imposing waits at any point where interference might occur. In doing so, 
they also may prevent some concurrency that would have been harmless to correctness. 
An alternative scheme, called optimistic concurrency control, is to presume that interfer­
ence between concurrent transactions is unlikely, and allow them to proceed without 
waiting. Then, watch for actual interference, and if it happens take some recovery action, 
for example aborting an interfering transaction and makaing it restart. (There is a popu­
lar tongue-in-cheek characterization of the difference: pessimistic = “ask first”, optimistic 
= “apologize later”.) The goal of optimistic concurrency control is to increase concur­
rency in situations where actual interference is rare. 
The system state history of Figure 9.27 suggests an opportunity to be optimistic. We 
could allow transactions to write values into the system state history in any order and at 
any time, but with the risk that some attempts to write may be met with the response 
“Sorry, that write would interfere with another transaction. You must abort, abandon 
this serialization position in the system state history, obtain a later serialization, and 
rerun your transaction from the beginning.” 
A specific example of this approach is the read-capture discipline. Under the read-cap­
ture discipline, there is an option, but not a requirement, of advance marking. 
Eliminating the requirement of advance marking has the advantage that a transaction 
does not need to predict the identity of every object it will update—it can discover the 
identity of those objects as it works. Instead of advance marking, whenever a transaction 
calls READ_CURRENT_VALUE, that procedure makes a mark at this thread’s position in the 
version history of the object it read. This mark tells potential version-inserters earlier in 
the serial ordering but arriving later in real time that they are no longer allowed to 
insert—they must abort and try again, using a later serial position in the version history. 
Had the prospective version inserter gotten there sooner, before the reader had left its 
mark, the new version would have been acceptable, and the reader would have instead 
waited for the version inserter to commit, and taken that new value instead of the earlier 
one. Read-capture gives the reader the power of extending validity of a version through 
intervening transactions, up to the reader’s own serialization position. This view of the 
situation is illustrated in Figure 9.32, which has the same version history as did Figure 
9.27. 
Saltzer & Kaashoek Ch. 9, p. 63 
June 24, 2009 12:26 am

## Page 66

9–64 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
Value of object at end of transaction 
1 
2 
3
4
5
6 
7 
A 
B 
0 
-10 
-6 
-12 
-2
HWM=2 
HWM=5 
HWM=6
HWM=3 
C 
0 
-4 
+2 
HWM=5
HWM=3 
D 
0 
-2 
Committed Committed 
Committed 
Aborted Committed 
Pending 
0 
+10 
+12 
0 
Outcome 
record
state 
Pending 
Conflict 
Changed value 
HWM=2 
HWM=6 
Conflict: Must abort! 
High-water mark
HWM= 
HWM=7 
HWM=7
HWM=4 
-4 
+2 
FIGURE 9.32 
Version history with high-water marks and the read-capture discipline. First, transaction 6, 
which is running concurrently with transaction 4, reads variable A, thus extending the high-
water mark of A to 6. Then, transaction 4 (which intends to transfer 2 from D to A) encounters 
a conﬂict when it tries to create a new version of A and discovers that the high-water mark of 
A has already been set by transaction 6, so 4 aborts and returns as transaction 7. Transaction 
7 retries transaction 4, extending the high-water marks of A and D to 7. 
The key property of read-capture is illustrated by an example in Figure 9.32. Trans­
action 4 was late in creating a new version of object A; by the time it tried to do the 
insertion, transaction 6 had already read the old value (+10) and thereby extended the 
validity of that old value to the beginning of transaction 6. Therefore, transaction 4 had 
to be aborted; it has been reincarnated to try again as transaction 7. In its new position 
as transaction 7, its first act is to read object D, extending the validity of its most recent 
committed value (zero) to the beginning of transaction 7. When it tries to read object A, 
it discovers that the most recent version is still uncommitted, so it must wait for transac­
tion 6 to either commit or abort. Note that if transaction 6 should now decide to create 
a new version of object C, it can do so without any problem, but if it should try to create 
a new version of object D, it would run into a conflict with the old, now extended version 
of D, and it would have to abort. 
Saltzer & Kaashoek Ch. 9, p. 64 
June 24, 2009 12:26 am

## Page 67

9.4 Before-or-After Atomicity I: Concepts 9–65  
FIGURE 9.33 
1
 procedure READ_CURRENT_VALUE (reference data_id, value, caller_id) 
2
 starting at end of data_id repeat until beginning 
3 
v ← previous version of data_id 
4 
if v.action_id ≥ caller_id then skip v 
5 
examine v.action_id.outcome_record 
6 
if PENDING then 
7 
WAIT for v.action_id to COMMIT or ABORT 
8 
if COMMITTED then 
9 
v.high_water_mark ← max(v.high_water_mark, caller_id) 
10 
return v.value 
11 
else skip v 
// Continue backward search 
12 
signal (“Tried to read an uninitialized variable!”) 
13 procedure NEW_VERSION (reference data_id, caller_id) 
14 
if (caller_id < data_id.high_water_mark) 
// Conflict with later reader. 
15 
or (caller_id < (LATEST_VERSION[data_id].action_id)) 
// Blind write conflict. 
16 
then ABORT this transaction and terminate this thread 
17 
add new version v at end of data_id 
18 
v.value ← 0 
19 
v.action_id ← caller_id 
20 procedure WRITE_VALUE (reference data_id, new_value, caller_id) 
21 
locate version v of data_id.history such that v.action_id = caller_id 
22 
(if not found, signal (“Tried to write without creating new version!”)) 
23 
v.value ← new_value 
Read-capture forms of READ_CURRENT_VALUE, NEW_VERSION, and WRITE_VALUE. 
Read-capture is relatively easy to implement in a version history system. We start, as 
shown in Figure 9.33, by adding a new step (at line 9) to READ_CURRENT_VALUE. This new 
step records with each data object a high-water mark—the serial number of the highest-
numbered transaction that has ever read a value from this object’s version history. The 
high-water mark serves as a warning to other transactions that have earlier serial numbers 
but are late in creating new versions. The warning is that someone later in the serial 
ordering has already read a version of this object from earlier in the ordering, so it is too 
late to create a new version now. We guarantee that the warning is heeded by adding a 
step to NEW_VERSION (at line 14), which checks the high-water mark for the object to be 
written, to see if any transaction with a higher serial number has already read the current 
version of the object. If not, we can create a new version without concern. But if the 
transaction serial number in the high-water mark is greater than this transaction’s own 
serial number, this transaction must abort, obtain a new, higher serial number, and start 
over again. 
Saltzer & Kaashoek Ch. 9, p. 65 
June 24, 2009 12:26 am

## Page 68

9–66 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
We have removed all constraints on the real-time sequence of the constituent steps of 
the concurrent transaction, so there is a possibility that a high-numbered transaction will 
create a new version of some object, and then later a low-numbered transaction will try 
to create a new version of the same object. Since our NEW_VERSION procedure simply tacks 
new versions on the end of the object history, we could end up with a history in the 
wrong order. The simplest way to avoid that mistake is to put an additional test in 
NEW_VERSION (at line 15), to ensure that every new version has a client serial number that 
is larger than the serial number of the next previous version. If not, NEW_VERSION aborts 
the transaction, just as if a read-capture conflict had occurred. (This test aborts only 
those transactions that perform conflicting blind writes, which are uncommon. If either 
of the conflicting transactions reads the value before writing it, the setting and testing of 
high_water_mark will catch and prevent the conflict.) 
The first question one must raise about this kind of algorithm is whether or not it 
actually works: is the result always the same as some serial ordering of the concurrent 
transactions? Because the read-capture discipline permits greater concurrency than does 
mark-point, the correctness argument is a bit more involved. The induction part of the 
argument goes as follows: 
1.  The WAIT for PENDING values in READ_CURRENT_VALUE ensures that if any pending 
transaction k < n has modiﬁed any value that is later read by transaction n, 
transaction n will wait for transaction k to commit or abort. 
2.  The setting of the high-water mark when transaction n calls READ_CURRENT_VALUE, 
together with the test of the high-water mark in NEW_VERSION ensures that if any 
transaction j < n tries to modify any value after transaction n has read that value, 
transaction j will abort and not modify that value. 
3.  Therefore, every value that READ_CURRENT_VALUE returns to transaction n will 
include the ﬁnal effect of all preceding transactions 1...n – 1. 
4.  Therefore, every transaction n will act as if it serially follows transaction n – 1. 
Optimistic coordination disciplines such as read-capture have the possibly surprising 
effect that something done by a transaction later in the serial ordering can cause a trans­
action earlier in the ordering to abort. This effect is the price of optimism; to be a good 
candidate for an optimistic discipline, an application probably should not have a lot of 
data interference. 
A subtlety of read-capture is that it is necessary to implement bootstrapping before-
or-after atomicity in the procedure NEW_VERSION, by adding a lock and calls to ACQUIRE and 
RELEASE because NEW_VERSION can now be called by two concurrent threads that happen 
to add new versions to the same variable at about the same time. In addition, NEW_VERSION 
must be careful to keep versions of the same variable in transaction order, so that the 
backward search performed by READ_CURRENT_VALUE works correctly. 
There is one final detail, an interaction with all-or-nothing recovery. High water 
marks should be stored in volatile memory, so that following a crash (which has the effect 
Saltzer & Kaashoek Ch. 9, p. 66  
June 24, 2009 12:26 am

## Page 69

9.4 Before-or-After Atomicity I: Concepts 9–67  
of aborting all pending transactions) the high water marks automatically disappear and 
thus don’t cause unnecessary aborts. 
9.4.4  Does Anyone Actually Use Version Histories for Before-or-After 
Atomicity? 
The answer is yes, but the most common use is in an application not likely to be encoun­
tered by a software specialist. Legacy processor architectures typically provide a limited 
number of registers (the “architectural registers”) in which the programmer can hold 
temporary results, but modern large scale integration technology allows space on a phys­
ical chip for many more physical registers than the architecture calls for. More registers 
generally allow better performance, especially in multiple-issue processor designs, which 
execute several sequential instructions concurrently whenever possible. To allow use of 
the many physical registers, a register mapping scheme known as register renaming imple­
ments a version history for the architectural registers. This version history allows 
instructions that would interfere with each other only because of a shortage of registers 
to execute concurrently. 
For example, Intel Pentium processors, which are based on the x86 instruction set 
architecture described in Section 5.7, have only eight architectural registers. The Pen­
tium 4 has 128 physical registers, and a register renaming scheme based on a circular 
reorder buffer. A reorder buffer resembles a direct hardware implementation of the pro­
cedures NEW_VERSION and WRITE_VALUE of Figure 9.29. As each instruction issues (which 
corresponds to BEGIN_TRANSACTION), it is assigned the next sequential slot in the reorder 
buffer. The slot is a map that maintains a correspondence between two numbers: the 
number of the architectural register that the programmer specified to hold the output 
value of the instruction, and the number of one of the 128 physical registers, the one that 
will actually hold that output value. Since machine instructions have just one output 
value, assigning a slot in the reorder buffer implements in a single step the effect of both 
NEW_OUTCOME_RECORD and NEW_VERSION. Similarly, when the instruction commits, it 
places its output in that physical register, thereby implementing WRITE_VALUE and COMMIT 
as a single step. 
Figure 9.34 illustrates register renaming with a reorder buffer. In the program 
sequence of that example, instruction n uses architectural register five to hold an output 
value that instruction n + 1 will use as an input. Instruction n + 2 loads architectural reg­
ister five from memory. Register renaming allows there to be two (or more) versions of 
register five simultaneously, one version (in physical register 42) containing a value for 
use by instructions n and n + 1 and the second version (in physical register 29) to be used 
by instruction n + 2. The performance benefit is that instruction n + 2 (and any later 
instructions that write into architectural register 5) can proceed concurrently with 
instructions n and n + 1. An instruction following instruction n + 2 that requires the new 
value in architectural register five as an input uses a hardware implementation of 
READ_CURRENT_VALUE to locate the most recent preceding mapping of architectural register 
five in the reorder buffer. In this case that most recent mapping is to physical register 29. 
Saltzer & Kaashoek Ch. 9, p. 67  
June 24, 2009 12:26 am

## Page 70

9–68 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
The later instruction then stalls, waiting for instruction n + 2 to write a value into phys­
ical register 29. Later instructions that reuse architectural register five for some purpose 
that does not require that version can proceed concurrently. 
Although register renaming is conceptually straightforward, the mechanisms that pre­
vent interference when there are dependencies between instructions tend to be more 
intricate than either of the mark-point or read-capture disciplines, so this description has 
been oversimplified. For more detail, the reader should consult a textbook on processor 
architecture, for example Computer Architecture, a Quantitative Approach, by Hennessy 
and Patterson [Suggestions for Further Reading 1.1.1]. 
The Oracle database management system offers several before-or-after atomicity 
methods, one of which it calls “serializable”, though the label may be a bit misleading. 
This method uses a before-or-after atomicity scheme that the database literature calls 
snapshot isolation. The idea is that when a transaction begins the system conceptually 
takes a snapshot of every committed value and the transaction reads all of its inputs from 
that snapshot. If two concurrent transactions (which might start with the same snapshot) 
modify the same variable, the first one to commit wins; the system aborts the other one 
with a “serialization error”. This scheme effectively creates a limited variant of a version 
architectural physical 
physical register file 
with 128 registers 
FIGURE 9.34 
n 
n + 1 
n + 2 
R5 
R4 
R5 
42 
61 
29 
three entries in the reorder buffer 
register 
register
instruction 
0 
127 
Example showing how a reorder buffer maps architectural register numbers to physical register 
numbers. The program sequence corresponding to the three entries is: 
n 
R5 ← R4 × R2 
// Write a result in register five. 
n + 1 R4  ← R5 + R1 
// Use result in register five. 
n + 2 R5  ← READ (117492) 
// Write content of a memory cell in register five. 
Instructions n and n + 2 both write into register R5, so R5 has two versions, with mappings to 
physical registers 42 and 29, respectively. Instruction n + 2 can thus execute concurrently with 
instructions n and n + 1. 
Saltzer & Kaashoek Ch. 9, p. 68 
June 24, 2009 12:26 am

## Page 71

9.5 Before-or-After Atomicity II: Pragmatics 9–69  
history that, in certain situations, does not always ensure that concurrent transactions are 
correctly coordinated. 
Another specialized variant implementation of version histories, known as transac­
tional memory, is a discipline for creating atomic actions from arbitrary instruction 
sequences that make multiple references to primary memory. Transactional memory was 
first suggested in 1993 and with widespread availability of multicore processors, has 
become the subject of quite a bit of recent research interest because it allows the applica­
tion programmer to use concurrent threads without having to deal with locks. The 
discipline is to mark the beginning of an instruction sequence that is to be atomic with 
a “begin transaction” instruction, direct all ensuing STORE instructions to a hidden copy 
of the data that concurrent threads cannot read, and at end of the sequence check to see 
that nothing read or written during the sequence was modified by some other transaction 
that committed first. If the check finds no such earlier modifications, the system com­
mits the transaction by exposing the hidden copies to concurrent threads; otherwise it 
discards the hidden copies and the transaction aborts. Because it defers all discovery of 
interference to the commit point this discipline is even more optimistic than the read-
capture discipline described in Section 9.4.3 above, so it is most useful in situations 
where interference between concurrent threads is possible but unlikely. Transactional 
memory has been experimentally implemented in both hardware and software. Hard­
ware implementations typically involve tinkering with either a cache or a reorder buffer 
to make it defer writing hidden copies back to primary memory until commit time, while 
software implementations create hidden copies of changed variables somewhere else in 
primary memory. As with instruction renaming, this description of transactional mem­
ory is somewhat oversimplified, and the interested reader should consult the literature 
for fuller explanations. 
Other software implementations of version histories for before-or-after atomicity 
have been explored primarily in research environments. Designers of database systems 
usually use locks rather than version histories because there is more experience in achiev­
ing high performance with locks. Before-or-after atomicity by using locks systematically 
is the subject of the next section of this chapter. 
9.5 Before-or-After Atomicity II: Pragmatics 
The previous section showed that a version history system that provides all-or-nothing 
atomicity can be extended to also provide before-or-after atomicity. When the all-or­
nothing atomicity design uses a log and installs data updates in cell storage, other, con­
current actions can again immediately see those updates, so we again need a scheme to 
provide before-or-after atomicity. When a system uses logs for all-or-nothing atomicity, 
it usually adopts the mechanism introduced in Chapter 5—locks—for before-or-after 
atomicity. However, as Chapter 5 pointed out, programming with locks is hazardous, 
and the traditional programming technique of debugging until the answers seem to be 
correct is unlikely to catch all locking errors. We now revisit locks, this time with the goal 
Saltzer & Kaashoek Ch. 9, p. 69 
June 24, 2009 12:26 am

## Page 72

9–70 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
of using them in stylized ways that allow us to develop arguments that the locks correctly 
implement before-or-after atomicity. 
9.5.1 Locks 
To review, a lock is a flag associated with a data object and set by an action to warn other, 
concurrent, actions not to read or write the object. Conventionally, a locking scheme 
involves two procedures: 
ACQUIRE (A.lock) 
marks a lock variable associated with object A as having been acquired. If the object is 
already acquired, ACQUIRE waits until the previous acquirer releases it. 
RELEASE (A.lock) 
unmarks the lock variable associated with A, perhaps ending some other action’s wait for 
that lock. For the moment, we assume that the semantics of a lock follow the single-
acquire protocol of Chapter 5: if two or more actions attempt to acquire a lock at about 
the same time, only one will succeed; the others must find the lock already acquired. In 
Section 9.5.4 we will consider some alternative protocols, for example one that permits 
several readers of a variable as long as there is no one writing it. 
The biggest problem with locks is that programming errors can create actions that do 
not have the intended before-or-after property. Such errors can open the door to races 
that, because the interfering actions are timing dependent, can make it extremely diffi­
cult to figure out what went wrong. Thus a primary goal is that coordination of 
concurrent transactions should be arguably correct. For locks, the way to achieve this 
goal is to follow three steps systematically: 
•  Develop a locking discipline that specifies which locks must be acquired and 
when. 
•   Establish a compelling line of reasoning that concurrent transactions that follow 
the discipline will have the before-or-after property. 
•   Interpose a  lock manager, a program that enforces the discipline, between the 
programmer and the ACQUIRE and RELEASE procedures. 
Many locking disciplines have been designed and deployed, including some that fail to 
correctly coordinate transactions (for an example, see exercise 9.5). We examine three 
disciplines that succeed. Each allows more concurrency than its predecessor, though even 
the best one is not capable of guaranteeing that concurrency is maximized. 
The first, and simplest, discipline that coordinates transactions correctly is the system-
wide lock. When the system first starts operation, it creates a single lockable variable 
named, for example, System, in volatile memory. The discipline is that every transaction 
must start with 
Saltzer & Kaashoek Ch. 9, p. 70  
June 24, 2009 12:26 am

## Page 73

9.5 Before-or-After Atomicity II: Pragmatics 9–71  
begin_transaction 
ACQUIRE (System.lock)  
…  
and every transaction must end with 
…  
RELEASE (System.lock)  
end_transaction 
A system can even enforce this discipline by including the ACQUIRE and RELEASE steps in 
the code sequence generated for begin_transaction and end_transaction, indepen­
dent of whether the result was COMMIT or ABORT. Any programmer who creates a new 
transaction then has a guarantee that it will run either before or after any other 
transactions. 
The systemwide lock discipline allows only one transaction to execute at a time. It 
serializes potentially concurrent transactions in the order that they call ACQUIRE. The sys­
temwide lock discipline is in all respects identical to the simple serialization discipline of 
Section 9.4. In fact, the simple serialization pseudocode 
id ← NEW_OUTCOME_RECORD ()  
preceding_id ← id - 1  
wait until preceding_id.outcome_record.value ≠ PENDING  
…  
COMMIT (id) [or ABORT (id)]  
and the systemwide lock invocation 
ACQUIRE (System.lock)  
…  
RELEASE (System.lock)  
are actually just two implementations of the same idea. 
As with simple serialization, systemwide locking restricts concurrency in cases where 
it doesn’t need to because it locks all data touched by every transaction. For example, if 
systemwide locking were applied to the funds TRANSFER program of Figure 9.16, only one 
transfer could occur at a time, even though any individual transfer involves only two out 
of perhaps several million accounts, so there would be many opportunities for concur­
rent, non-interfering transfers. Thus there is an interest in developing less restrictive 
locking disciplines. The starting point is usually to employ a finer lock granularity: lock 
smaller objects, such as individual data records, individual pages of data records, or even 
fields within records. The trade-offs in gaining concurrency are first, that when there is 
more than one lock, more time is spent acquiring and releasing locks and second, cor­
rectness arguments become more complex. One hopes that the performance gain from 
concurrency exceeds the cost of acquiring and releasing the multiple locks. Fortunately, 
there are at least two other disciplines for which correctness arguments are feasible, simple 
locking and two-phase locking. 
Saltzer & Kaashoek Ch. 9, p. 71 
June 24, 2009 12:26 am

## Page 74

9–72 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
9.5.2 Simple Locking 
The second locking discipline, known as simple locking, is similar in spirit to, though not 
quite identical with, the mark-point discipline. The simple locking discipline has two 
rules. First, each transaction must acquire a lock for every shared data object it intends 
to read or write before doing any actual reading and writing. Second, it may release its 
locks only after the transaction installs its last update and commits or completely restores 
the data and aborts. Analogous to the mark point, the transaction has what is called a lock 
point: the first instant at which it has acquired all of its locks. The collection of locks it 
has acquired when it reaches its lock point is called its lock set. A lock manager can enforce 
simple locking by requiring that each transaction supply its intended lock set as an argu­
ment to the begin_transaction operation, which acquires all of the locks of the lock set, 
if necessary waiting for them to become available. The lock manager can also interpose 
itself on all calls to read data and to log changes, to verify that they refer to variables that 
are in the lock set. The lock manager also intercepts the call to commit or abort (or, if 
the application uses roll-forward recovery, to log an END record) at which time it auto­
matically releases all of the locks of the lock set. 
The simple locking discipline correctly coordinates concurrent transactions. We can 
make that claim using a line of argument analogous to the one used for correctness of the 
mark-point discipline. Imagine that an all-seeing outside observer maintains an ordered 
list to which it adds each transaction identifier as soon as the transaction reaches its lock 
point and removes it from the list when it begins to release its locks. Under the simple 
locking discipline each transaction has agreed not to read or write anything until that 
transaction has been added to the observer’s list. We also know that all transactions that 
precede this one in the list must have already passed their lock point. Since no data object 
can appear in the lock sets of two transactions, no data object in any transaction’s lock 
set appears in the lock set of the transaction preceding it in the list, and by induction to 
any transaction earlier in the list. Thus all of this transaction’s input values are the same 
as they will be when the preceding transaction in the list commits or aborts. The same 
argument applies to the transaction before the preceding one, so all inputs to any trans­
action are identical to the inputs that would be available if all the transactions ahead of 
it in the list ran serially, in the order of the list. Thus the simple locking discipline ensures 
that this transaction runs completely after the preceding one and completely before the 
next one. Concurrent transactions will produce results as if they had been serialized in 
the order that they reached their lock points. 
As with the mark-point discipline, simple locking can miss some opportunities for 
concurrency. In addition, the simple locking discipline creates a problem that can be sig­
nificant in some applications. Because it requires the transaction to acquire a lock on 
every shared object that it will either read or write (recall that the mark-point discipline 
requires marking only of shared objects that the transaction will write), applications that 
discover which objects need to be read by reading other shared data objects have no alter­
native but to lock every object that they might need to read. To the extent that the set of 
objects that an application might need to read is larger than the set for which it eventually 
Saltzer & Kaashoek Ch. 9, p. 72 
June 24, 2009 12:26 am

## Page 75

9.5 Before-or-After Atomicity II: Pragmatics 9–73  
does read, the simple locking discipline can interfere with opportunities for concurrency. 
On the other hand, when the transaction is straightforward (such as the TRANSFER trans­
action of Figure 9.16, which needs to lock only two records, both of which are known at 
the outset) simple locking can be effective. 
9.5.3 Two-Phase Locking 
The third locking discipline, called two-phase locking, like the read-capture discipline, 
avoids the requirement that a transaction must know in advance which locks to acquire. 
Two-phase locking is widely used, but it is harder to argue that it is correct. The two-
phase locking discipline allows a transaction to acquire locks as it proceeds, and the trans­
action may read or write a data object as soon as it acquires a lock on that object. The 
primary constraint is that the transaction may not release any locks until it passes its lock 
point. Further, the transaction can release a lock on an object that it only reads any time 
after it reaches its lock point if it will never need to read that object again, even to abort. 
The name of the discipline comes about because the number of locks acquired by a trans­
action monotonically increases up to the lock point (the first phase), after which it 
monotonically decreases (the second phase). Just as with simple locking, two-phase lock­
ing orders concurrent transactions so that they produce results as if they had been 
serialized in the order they reach their lock points. A lock manager can implement two-
phase locking by intercepting all calls to read and write data; it acquires a lock (perhaps 
having to wait) on the first use of each shared variable. As with simple locking, it then 
holds the locks until it intercepts the call to commit, abort, or log the END record of the 
transaction, at which time it releases them all at once. 
The extra flexibility of two-phase locking makes it harder to argue that it guarantees 
before-or-after atomicity. Informally, once a transaction has acquired a lock on a data 
object, the value of that object is the same as it will be when the transaction reaches its 
lock point, so reading that value now must yield the same result as waiting till then to 
read it. Furthermore, releasing a lock on an object that it hasn’t modified must be harm­
less if this transaction will never look at the object again, even to abort. A formal 
argument that two-phase locking leads to correct before-or-after atomicity can be found 
in most advanced texts on concurrency control and transactions. See, for example, Trans­
action Processing, by Gray and Reuter [Suggestions for Further Reading 1.1.5]. 
The two-phase locking discipline can potentially allow more concurrency than the 
simple locking discipline, but it still unnecessarily blocks certain serializable, and there­
fore correct, action orderings. For example, suppose transaction T1 reads X and writes Y, 
while transaction T2 just does a (blind) write to Y. Because the lock sets of T1 and T2 
intersect at variable Y, the two-phase locking discipline will force transaction T2 to run 
either completely before or completely after T1. But the sequence 
T1: READ X  
T2: WRITE Y  
T1: WRITE Y  
Saltzer & Kaashoek Ch. 9, p. 73 
June 24, 2009 12:26 am

## Page 76

9–74 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
in which the write of T2 occurs between the two steps of T1, yields the same result as 
running T2 completely before T1, so the result is always correct, even though this 
sequence would be prevented by two-phase locking. Disciplines that allow all possible 
concurrency while at the same time ensuring before-or-after atomicity are quite difficult 
to devise. (Theorists identify the problem as NP-complete.) 
There are two interactions between locks and logs that require some thought: (1) 
individual transactions that abort, and (2) system recovery. Aborts are the easiest to deal 
with. Since we require that an aborting transaction restore its changed data objects to 
their original values before releasing any locks, no special account need be taken of 
aborted transactions. For purposes of before-or-after atomicity they look just like com­
mitted transactions that didn’t change anything. The rule about not releasing any locks 
on modified data before the end of the transaction is essential to accomplishing an abort. 
If a lock on some modified object were released, and then the transaction decided to 
abort, it might find that some other transaction has now acquired that lock and changed 
the object again. Backing out an aborted change is likely to be impossible unless the locks 
on modified objects have been held. 
The interaction between log-based recovery and locks is less obvious. The question is 
whether locks themselves are data objects for which changes should be logged. To ana­
lyze this question, suppose there is a system crash. At the completion of crash recovery 
there should be no pending transactions because any transactions that were pending at 
the time of the crash should have been rolled back by the recovery procedure, and recov­
ery does not allow any new transactions to begin until it completes. Since locks exist only 
to coordinate pending transactions, it would clearly be an error if there were locks still 
set when crash recovery is complete. That observation suggests that locks belong in vol­
atile storage, where they will automatically disappear on a crash, rather than in non­
volatile storage, where the recovery procedure would have to hunt them down to release 
them. The bigger question, however, is whether or not the log-based recovery algorithm 
will construct a correct system state—correct in the sense that it could have arisen from 
some serial ordering of those transactions that committed before the crash. 
Continue to assume that the locks are in volatile memory, and at the instant of a crash 
all record of the locks is lost. Some set of transactions—the ones that logged a BEGIN 
record but have not yet logged an END record—may not have been completed. But we 
know that the transactions that were not complete at the instant of the crash had non-
overlapping lock sets at the moment that the lock values vanished. The recovery algo­
rithm of Figure 9.23 will systematically UNDO or REDO installs for the incomplete 
transactions, but every such UNDO or REDO must modify a variable whose lock was in some 
transaction’s lock set at the time of the crash. Because those lock sets must have been 
non-overlapping, those particular actions can safely be redone or undone without con­
cern for before-or-after atomicity during recovery. Put another way, the locks created a 
particular serialization of the transactions and the log has captured that serialization. 
Since RECOVER performs UNDO actions in reverse order as specified in the log, and it per­
forms REDO actions in forward order, again as specified in the log, RECOVER reconstructs 
exactly that same serialization. Thus even a recovery algorithm that reconstructs the 
Saltzer & Kaashoek Ch. 9, p. 74 
June 24, 2009 12:26 am

## Page 77

9.5 Before-or-After Atomicity II: Pragmatics 9–75  
entire database from the log is guaranteed to produce the same serialization as when the 
transactions were originally performed. So long as no new transactions begin until recov­
ery is complete, there is no danger of miscoordination, despite the absence of locks 
during recovery. 
9.5.4 Performance Optimizations 
Most logging-locking systems are substantially more complex than the description so far 
might lead one to expect. The complications primarily arise from attempts to gain per­
formance. In Section 9.3.6 we saw how buffering of disk I/O in a volatile memory cache, 
to allow reading, writing, and computation to go on concurrently, can complicate a log­
ging system. Designers sometimes apply two performance-enhancing complexities to 
locking systems: physical locking and adding lock compatibility modes. 
A performance-enhancing technique driven by buffering of disk I/O and physical 
media considerations is to choose a particular lock granularity known as physical locking. 
If a transaction makes a change to a six-byte object in the middle of a 1000-byte disk 
sector, or to a 1500-byte object that occupies parts of two disk sectors, there is a question 
about which “variable” should be locked: the object, or the disk sector(s)? If two concur­
rent threads make updates to unrelated data objects that happen to be stored in the same 
disk sector, then the two disk writes must be coordinated. Choosing the right locking 
granularity can make a big performance difference. 
Locking application-defined objects without consideration of their mapping to phys­
ical disk sectors is appealing because it is understandable to the application writer. For 
that reason, it is usually called logical locking. In addition, if the objects are small, it appar­
ently allows more concurrency: if another transaction is interested in a different object 
that is in the same disk sector, it could proceed in parallel. However, a consequence of 
logical locking is that logging must also be done on the same logical objects. Different 
parts of the same disk sector may be modified by different transactions that are running 
concurrently, and if one transaction commits but the other aborts neither the old nor the 
new disk sector is the correct one to restore following a crash; the log entries must record 
the old and new values of the individual data objects that are stored in the sector. Finally, 
recall that a high-performance logging system with a cache must, at commit time, force 
the log to disk and keep track of which objects in the cache it is safe to write to disk with­
out violating the write-ahead log protocol. So logical locking with small objects can 
escalate cache record-keeping. 
Backing away from the details, high-performance disk management systems typically 
require that the argument of a PUT call be a block whose size is commensurate with the 
size of a disk sector. Thus the real impact of logical locking is to create a layer between 
the application and the disk management system that presents a logical, rather than a 
physical, interface to its transaction clients; such things as data object management and 
garbage collection within disk sectors would go into this layer. The alternative is to tailor 
the logging and locking design to match the native granularity of the disk management 
system. Since matching the logging and locking granularity to the disk write granularity 
Saltzer & Kaashoek Ch. 9, p. 75 
June 24, 2009 12:26 am

## Page 78

9–76 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
can reduce the number of disk operations, both logging changes to and locking blocks 
that correspond to disk sectors rather than individual data objects is a common practice. 
Another performance refinement appears in most locking systems: the specification 
of lock compatibility modes. The idea is that when a transaction acquires a lock, it can 
specify what operation (for example, READ or WRITE) it intends to perform on the locked 
data item. If that operation is compatible—in the sense that the result of concurrent 
transactions is the same as some serial ordering of those transactions—then this transac­
tion can be allowed to acquire a lock even though some other transaction has already 
acquired a lock on that same data object. 
The most common example involves replacing the single-acquire locking protocol 
with the multiple-reader, single-writer protocol. According to this protocol, one can allow 
any number of readers to simultaneously acquire read-mode locks for the same object. 
The purpose of a read-mode lock is to ensure that no other thread can change the data 
while the lock is held. Since concurrent readers do not present an update threat, it is safe 
to allow any number of them. If another transaction needs to acquire a write-mode lock 
for an object on which several threads already hold read-mode locks, that new transaction 
will have to wait for all of the readers to release their read-mode locks. There are many 
applications in which a majority of data accesses are for reading, and for those applica­
tions the provision of read-mode lock compatibility can reduce the amount of time spent 
waiting for locks by orders of magnitude. At the same time, the scheme adds complexity, 
both in the mechanics of locking and also in policy issues, such as what to do if, while a 
prospective writer is waiting for readers to release their read-mode locks, another thread 
calls to acquire a read-mode lock. If there is a steady stream of arriving readers, a writer 
could be delayed indefinitely. 
This description of performance optimizations and their complications is merely 
illustrative, to indicate the range of opportunities and kinds of complexity that they 
engender; there are many other performance-enhancement techniques, some of which 
can be effective, and others that are of dubious value; most have different values depend­
ing on the application. For example, some locking disciplines compromise before-or­
after atomicity by allowing transactions to read data values that are not yet committed. 
As one might expect, the complexity of reasoning about what can or cannot go wrong in 
such situations escalates. If a designer intends to implement a system using performance 
enhancements such as buffering, lock compatibility modes, or compromised before-or­
after atomicity, it would be advisable to study carefully the book by Gray and Reuter, as 
well as existing systems that implement similar enhancements. 
9.5.5 Deadlock; Making Progress 
Section 5.2.5 of Chapter 5 introduced the emergent problem of deadlock, the wait-for 
graph as a way of analyzing deadlock, and lock ordering as a way of preventing deadlock. 
With transactions and the ability to undo individual actions or even abort a transaction 
completely we now have more tools available to deal with deadlock, so it is worth revis­
iting that discussion. 
Saltzer & Kaashoek Ch. 9, p. 76 
June 24, 2009 12:26 am

## Page 79

9.5 Before-or-After Atomicity II: Pragmatics 9–77  
The possibility of deadlock is an inevitable consequence of using locks to coordinate 
concurrent activities. Any number of concurrent transactions can get hung up in a dead­
lock, either waiting for one another, or simply waiting for a lock to be released by some 
transaction that is already deadlocked. Deadlock leaves us a significant loose end: cor­
rectness arguments ensure us that any transactions that complete will produce results as 
though they were run serially, but they say nothing about whether or not any transaction 
will ever complete. In other words, our system may ensure correctness, in the sense that 
no wrong answers ever come out, but it does not ensure progress—no answers may come 
out at all. 
As with methods for concurrency control, methods for coping with deadlock can also 
be described as pessimistic or optimistic. Pessimistic methods take a priori action to pre­
vent deadlocks from happening. Optimistic methods allow concurrent threads to 
proceed, detect deadlocks if they happen, and then take action to fix things up. Here are 
some of the most popular methods: 
1.  Lock ordering (pessimistic). As suggested in Chapter 5, number the locks uniquely, 
and require that transactions acquire locks in ascending numerical order. With this 
plan, when a transaction encounters an already-acquired lock, it is always safe to 
wait for it, since the transaction that previously acquired it cannot be waiting for 
any locks that this transaction has already acquired—all those locks are lower in 
number than this one. There is thus a guarantee that somewhere, at least one 
transaction (the one holding the highest-numbered lock) can always make 
progress. When that transaction ﬁnishes, it will release all of its locks, and some 
other transaction will become the one that is guaranteed to be able to make 
progress. A generalization of lock ordering that may eliminate some unnecessary 
waits is to arrange the locks in a lattice and require that they be acquired in some 
lattice traversal order. The trouble with lock ordering, as with simple locking, is 
that some applications may not be able to predict all of the locks they need before 
acquiring the ﬁrst one. 
2.  Backing out (optimistic): An elegant strategy devised by Andre Bensoussan in 1966 
allows a transaction to acquire locks in any order, but if it encounters an already-
acquired lock with a number lower than one it has previously acquired itself, the 
transaction must back up (in terms of this chapter, UNDO previous actions) just far 
enough to release its higher-numbered locks, wait for the lower-numbered lock to 
become available, acquire that lock, and then REDO the backed-out actions. 
3.  Timer expiration (optimistic). When a new transaction begins, the lock manager 
sets an interrupting timer to a value somewhat greater than the time it should take 
for the transaction to complete. If a transaction gets into a deadlock, its timer will 
expire, at which point the system aborts that transaction, rolling back its changes 
and releasing its locks in the hope that the other transactions involved in the 
deadlock may be able to proceed. If not, another one will time out, releasing 
further locks. Timing out deadlocks is effective, though it has the usual defect: it 
Saltzer & Kaashoek Ch. 9, p. 77  
June 24, 2009 12:26 am

## Page 80

9–78 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
is difﬁcult to choose a suitable timer value that keeps things moving along but also 
accommodates normal delays and variable operation times. If the environment or 
system load changes, it may be necessary to readjust all such timer values, an 
activity that can be a real nuisance in a large system. 
4.  Cycle detection (optimistic). Maintain, in the lock manager, a wait-for graph (as 
described in Section 5.2.5) that shows which transactions have acquired which 
locks and which transactions are waiting for which locks. Whenever another 
transaction tries to acquire a lock, ﬁnds it is already locked, and proposes to wait, 
the lock manager examines the graph to see if waiting would produce a cycle, and 
thus a deadlock. If it would, the lock manager selects some cycle member to be a 
victim, and unilaterally aborts that transaction, so that the others may continue. 
The aborted transaction then retries in the hope that the other transactions have 
made enough progress to be out of the way and another deadlock will not occur. 
When a system uses lock ordering, backing out, or cycle detection, it is common to also 
set a timer as a safety net because a hardware failure or a programming error such as an 
endless loop can create a progress-blocking situation that none of the deadlock detection 
methods can catch. 
Since a deadlock detection algorithm can introduce an extra reason to abort a trans­
action, one can envision pathological situations where the algorithm aborts every 
attempt to perform some particular transaction, no matter how many times its invoker 
retries. Suppose, for example, that two threads named Alphonse and Gaston get into a 
deadlock trying to acquire locks for two objects named Apple and Banana: Alphonse 
acquires the lock for Apple, Gaston acquires the lock for Banana, Alphonse tries to 
acquire the lock for Banana and waits, then Gaston tries to acquire the lock for Apple 
and waits, creating the deadlock. Eventually, Alphonse times out and begins rolling back 
updates in preparation for releasing locks. Meanwhile, Gaston times out and does the 
same thing. Both restart, and they get into another deadlock, with their timers set to 
expire exactly as before, so they will probably repeat the sequence forever. Thus we still 
have no guarantee of progress. This is the emergent property that Chapter 5 called live-
lock, since formally no deadlock ever occurs and both threads are busy doing something 
that looks superficially useful. 
One way to deal with livelock is to apply a randomized version of a technique familiar 
from Chapter 7[on-line]: exponential random backoff. When a timer expiration leads to 
an abort, the lock manager, after clearing the locks, delays that thread for a random 
length of time, chosen from some starting interval, in the hope that the randomness will 
change the relative timing of the livelocked transactions enough that on the next try one 
will succeed and then the other can then proceed without interference. If the transaction 
again encounters interference, it tries again, but on each retry not only does the lock 
manager choose a new random delay, but it also increases the interval from which the 
delay is chosen by some multiplicative constant, typically 2. Since on each retry there is 
an increased probability of success, one can push this probability as close to unity as 
desired by continued retries, with the expectation that the interfering transactions will 
Saltzer & Kaashoek Ch. 9, p. 78  
June 24, 2009 12:26 am

## Page 81

9.6 Atomicity across Layers and Multiple Sites 9–79  
eventually get out of one another’s way. A useful property of exponential random backoff 
is that if repeated retries continue to fail it is almost certainly an indication of some 
deeper problem—perhaps a programming mistake or a level of competition for shared 
variables that is intrinsically so high that the system should be redesigned. 
The design of more elaborate algorithms or programming disciplines that guarantee 
progress is a project that has only modest potential payoff, and an end-to-end argument 
suggests that it may not be worth the effort. In practice, systems that would have frequent 
interference among transactions are not usually designed with a high degree of concur­
rency anyway. When interference is not frequent, simple techniques such as safety-net 
timers and exponential random backoff not only work well, but they usually must be 
provided anyway, to cope with any races or programming errors such as endless loops 
that may have crept into the system design or implementation. Thus a more complex 
progress-guaranteeing discipline is likely to be redundant, and only rarely will it get a 
chance to promote progress. 
9.6 Atomicity across Layers and Multiple Sites 
There remain some important gaps in our exploration of atomicity. First, in a layered 
system, a transaction implemented in one layer may consist of a series of component 
actions of a lower layer that are themselves atomic. The question is how the commitment 
of the lower-layer transactions should relate to the commitment of the higher layer trans­
action. If the higher-layer transaction decides to abort, the question is what to do about 
lower-layer transactions that may have already committed. There are two possibilities: 
•  Reverse the effect of any committed lower-layer transactions with an UNDO 
action. This technique requires that the results of the lower-layer transactions be 
visible only within the higher-layer transaction. 
•  Somehow delay commitment of the lower-layer transactions and arrange that 
they actually commit at the same time that the higher-layer transaction commits. 
Up to this point, we have assumed the first possibility. In this section we explore the sec­
ond one. 
Another gap is that, as described so far, our techniques to provide atomicity all 
involve the use of shared variables in memory or storage (for example, pointers to the lat­
est version, outcome records, logs, and locks) and thus implicitly assume that the 
composite actions that make up a transaction all occur in close physical proximity. When 
the composing actions are physically separated, communication delay, communication 
reliability, and independent failure make atomicity both more important and harder to 
achieve. 
We will edge up on both of these problems by first identifying a common subprob­
lem: implementing nested transactions. We will then extend the solution to the nested 
transaction problem to create an agreement protocol, known as two-phase commit, that 
Saltzer & Kaashoek Ch. 9, p. 79  
June 24, 2009 12:26 am

## Page 82

9–80 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
FIGURE 9.35 
procedure PAY_INTEREST (reference account) 
if account.balance > 0 then 
interest = account.balance * 0.05 
TRANSFER (bank, account, interest) 
else 
interest = account.balance * 0.15 
TRANSFER (account, bank, interest) 
procedure MONTH_END_INTEREST:() 
for A ← each customer_account do 
PAY_INTEREST (A) 
An example of two procedures, one of which calls the other, yet each should be individually 
atomic. 
coordinates commitment of lower-layer transactions. We can then extend the two-phase 
commit protocol, using a specialized form of remote procedure call, to coordinate steps 
that must be carried out at different places. This sequence is another example of boot­
strapping; the special case that we know how to handle is the single-site transaction and 
the more general problem is the multiple-site transaction. As an additional observation, 
we will discover that multiple-site transactions are quite similar to, but not quite the 
same as, the dilemma of the two generals. 
9.6.1 Hierarchical Composition of Transactions 
We got into the discussion of transactions by considering that complex interpreters are 
engineered in layers, and that each layer should implement atomic actions for its next-
higher, client layer. Thus transactions are nested, each one typically consisting of multi­
ple lower-layer transactions. This nesting requires that some additional thought be given 
to the mechanism of achieving atomicity. 
Consider again a banking example. Suppose that the TRANSFER procedure of Section 
9.1.5 is available for moving funds from one account to another, and it has been imple­
mented as a transaction. Suppose now that we wish to create the two application 
procedures of Figure 9.35. The first procedure, PAY_INTEREST, invokes TRANSFER to move 
an appropriate amount of money from or to an internal account named bank, the direc­
tion and rate depending on whether the customer account balance is positive or negative. 
The second procedure, MONTH_END_INTEREST, fulfills the bank’s intention to pay (or 
extract) interest every month on every customer account by iterating through the 
accounts and invoking PAY_INTEREST on each one. 
It would probably be inappropriate to have two invocations of MONTH_END_INTEREST 
running at the same time, but it is likely that at the same time that MONTH_END_INTEREST 
is running there are other banking activities in progress that are also invoking TRANSFER. 
Saltzer & Kaashoek Ch. 9, p. 80 
June 24, 2009 12:26 am

## Page 83

9.6 Atomicity across Layers and Multiple Sites 9–81  
It is also possible that the for each statement inside MONTH_END_INTEREST actually runs 
several instances of its iteration (and thus of PAY_INTEREST) concurrently. Thus we have a 
need for three layers of transactions. The lowest layer is the TRANSFER procedure, in which 
debiting of one account and crediting of a second account must be atomic. At the next 
higher layer, the procedure PAY_INTEREST should be executed atomically, to ensure that 
some concurrent TRANSFER transaction doesn’t change the balance of the account between 
the positive/negative test and the calculation of the interest amount. Finally, the proce­
dure MONTH_END_INTEREST should be a transaction, to ensure that some concurrent 
TRANSFER transaction does not move money from an account A to an account B between 
the interest-payment processing of those two accounts, since such a transfer could cause 
the bank to pay interest twice on the same funds. Structurally, an invocation of the TRANS­
FER procedure is nested inside PAY_INTEREST, and one or more concurrent invocations of 
PAY_INTEREST are nested inside MONTH_END_INTEREST. 
The reason nesting is a potential problem comes from a consideration of the commit 
steps of the nested transactions. For example, the commit point of the TRANSFER transac­
tion would seem to have to occur either before or after the commit point of the 
PAY_INTEREST transaction, depending on where in the programming of PAY_INTEREST we 
place its commit point. Yet either of these positions will cause trouble. If the TRANSFER 
commit occurs in the pre-commit phase of PAY_INTEREST then if there is a system crash 
PAY_INTEREST will not be able to back out as though it hadn’t tried to operate because the 
values of the two accounts that TRANSFER changed may have already been used by concur­
rent transactions to make payment decisions. But if the TRANSFER commit does not occur 
until the post-commit phase of PAY_INTEREST, there is a risk that the transfer itself can not 
be completed, for example because one of the accounts is inaccessible. The conclusion is 
that somehow the commit point of the nested transaction should coincide with the com­
mit point of the enclosing transaction. A slightly different coordination problem applies 
to MONTH_END_INTEREST: no TRANSFERs by other transactions should occur while it runs 
(that is, it should run either before or after any concurrent TRANSFER transactions), but it 
must be able to do multiple TRANSFERs itself, each time it invokes PAY_INTEREST, and its own 
possibly concurrent transfer actions must be before-or-after actions, since they all involve 
the account named “bank”. 
Suppose for the moment that the system provides transactions with version histories. 
We can deal with nesting problems by extending the idea of an outcome record: we allow 
outcome records to be organized hierarchically. Whenever we create a nested transaction, 
we record in its outcome record both the initial state (PENDING) of the new transaction and 
the identifier of the enclosing transaction. The resulting hierarchical arrangement of out­
come records then exactly reflects the nesting of the transactions. A top-layer outcome 
record would contain a flag to indicate that it is not nested inside any other transaction. 
When an outcome record contains the identifier of a higher-layer transaction, we refer 
to it as a dependent outcome record, and the record to which it refers is called its superior. 
The transactions, whether nested or enclosing, then go about their business, and 
depending on their success mark their own outcome records COMMITTED or ABORTED, as 
usual. However, when READ_CURRENT_VALUE (described in Section 9.4.2) examines the sta-
Saltzer & Kaashoek Ch. 9, p. 81 
June 24, 2009 12:26 am

## Page 84

9–82 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
tus of a version to see whether or not the transaction that created it is COMMITTED, it must 
additionally check to see if the outcome record contains a reference to a superior out­
come record. If so, it must follow the reference and check the status of the superior. If 
that record says that it, too, is COMMITTED, it must continue following the chain upward, 
if necessary all the way to the highest-layer outcome record. The transaction in question 
is actually COMMITTED only if all the records in the chain are in the COMMITTED state. If any 
record in the chain is ABORTED, this transaction is actually ABORTED, despite the COMMITTED 
claim in its own outcome record. Finally, if neither of those situations holds, then there 
must be one or more records in the chain that are still PENDING. The outcome of this trans­
action remains PENDING until those records become COMMITTED or ABORTED. Thus the 
outcome of an apparently-COMMITTED dependent outcome record actually depends on the 
outcomes of all of its ancestors. We can describe this situation by saying that, until all its 
ancestors commit, this lower-layer transaction is sitting on a knife-edge, at the point of 
committing but still capable of aborting if necessary. For purposes of discussion we will 
identify this situation as a distinct virtual state of the outcome record and the transaction, 
by saying that the transaction is tentatively committed. 
This hierarchical arrangement has several interesting programming consequences. If 
a nested transaction has any post-commit steps, those steps cannot proceed until all of 
the hierarchically higher transactions have committed. For example, if one of the nested 
transactions opens a cash drawer when it commits, the sending of the release message to 
the cash drawer must somehow be held up until the highest-layer transaction determines 
its outcome. 
This output visibility consequence is only one example of many relating to the tenta­
tively committed state. The nested transaction, having declared itself tentatively 
committed, has renounced the ability to abort—the decision is in someone else’s hands. 
It must be able to run to completion or to abort, and it must be able to maintain the ten­
tatively committed state indefinitely. Maintaining the ability to go either way can be 
awkward, since the transaction may be holding locks, keeping pages in memory or tapes 
mounted, or reliably holding on to output messages. One consequence is that a designer 
cannot simply take any arbitrary transaction and blindly use it as a nested component of 
a larger transaction. At the least, the designer must review what is required for the nested 
transaction to maintain the tentatively committed state. 
Another, more complex, consequence arises when one considers possible interactions 
among different transactions that are nested within the same higher-layer transaction. 
Consider our earlier example of TRANSFER transactions that are nested inside PAY_INTEREST, 
which in turn is nested inside MONTH_END_INTEREST. Suppose that the first time that 
MONTH_END_INTEREST invokes PAY_INTEREST, that invocation commits, thus moving into the 
tentatively committed state, pending the outcome of MONTH_END_INTEREST. Then 
MONTH_END_INTEREST invokes PAY_INTEREST on a second bank account. PAY_INTEREST needs 
to be able to read as input data the value of the bank’s own interest account, which is a 
pending result of the previous, tentatively COMMITTED, invocation of PAY_INTEREST. The 
READ_CURRENT_VALUE algorithm, as implemented in Section 9.4.2, doesn’t distinguish 
between reads arising within the same group of nested transactions and reads from some 
Saltzer & Kaashoek Ch. 9, p. 82 
June 24, 2009 12:26 am

## Page 85

9.6 Atomicity across Layers and Multiple Sites 9–83  
completely unrelated transaction. Figure 9.36 illustrates the situation. If the test in 
READ_CURRENT_VALUE for committed values is extended by simply following the ancestry of 
the outcome record controlling the latest version, it will undoubtedly force the second 
invocation of PAY_INTEREST to wait pending the final outcome of the first invocation of 
PAY_INTEREST. But since the outcome of that first invocation depends on the outcome of 
MONTH_END_INTEREST 
outcome: 
superior: 
PENDING 
outcome: 
superior: 
PAY_INTEREST1  (1st invocation) 
COMMITTED 
MONTH_END_INTEREST 
outcome: 
superior: 
TRANSFER1 
COMMITTED 
PAY_INTEREST1 
OK for TRANSFER2 
none 
outcome: 
superior: 
PAY_INTEREST2  (2nd invocation) 
PENDING 
MONTH_END_INTEREST 
outcome: 
superior: 
TRANSFER2 
PENDING 
PAY_INTEREST2 
to read? 
creator: TRANSFER1 
newest version of 
account bank
FIGURE 9.36 
Transaction TRANSFER2, nested in transaction PAY_INTEREST2, which is nested in transaction 
MONTH_END_INTEREST, wants to read the current value of account bank. But bank was last writ­
ten by transaction TRANSFER1, which is nested in COMMITTED transaction PAY_INTEREST1, which is 
nested in still-PENDING transaction MONTH_END_INTEREST. Thus this version of bank is actually 
PENDING, rather than COMMITTED as one might conclude by looking only at the outcome of 
TRANSFER1. However, TRANSFER1 and TRANSFER2 share a common ancestor (namely, 
MONTH_END_INTEREST), and the chain of transactions leading from bank to that common ances­
tor is completely committed, so the read of bank can—and to avoid a deadlock, must—be 
allowed. 
Saltzer & Kaashoek Ch. 9, p. 83 
June 24, 2009 12:26 am

## Page 86

9–84 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
MONTH_END_INTEREST, and the outcome of MONTH_END_INTEREST currently depends on the 
success of the second invocation of PAY_INTEREST, we have a built-in cycle of waits that at 
best can only time out and abort. 
Since blocking the read would be a mistake, the question of when it might be OK to 
permit reading of data values created by tentatively COMMITTED transactions requires some 
further thought. The before-or-after atomicity requirement is that no update made by a 
tentatively COMMITTED transaction should be visible to any transaction that would survive 
if for some reason the tentatively COMMITTED transaction ultimately aborts. Within that 
constraint, updates of tentatively COMMITTED transactions can freely be passed around. We 
can achieve that goal in the following way: compare the outcome record ancestry of the 
transaction doing the read with the ancestry of the outcome record that controls the ver­
sion to be read. If these ancestries do not merge (that is, there is no common ancestor) 
then the reader must wait for the version’s ancestry to be completely committed. If they 
do merge and all the transactions in the ancestry of the data version that are below the 
point of the merge are tentatively committed, no wait is necessary. Thus, in Figure 9.36, 
MONTH_END_INTEREST might be running the two (or more) invocations of PAY_INTEREST con­
currently. Each invocation will call CREATE_NEW_VERSION as part of its plan to update the 
value of account “bank”, thereby establishing a serial order of the invocations. When 
later invocations of PAY_INTEREST call READ_CURRENT_VALUE to read the value of account 
“bank”, they will be forced to wait until all earlier invocations of PAY_INTEREST decide 
whether to commit or abort. 
9.6.2 Two-Phase Commit 
Since a higher-layer transaction can comprise several lower-layer transactions, we can 
describe the commitment of a hierarchical transaction as involving two distinct phases. 
In the first phase, known variously as the preparation or voting phase, the higher-layer 
transaction invokes some number of distinct lower-layer transactions, each of which 
either aborts or, by committing, becomes tentatively committed. The top-layer transac­
tion evaluates the situation to establish that all (or enough) of the lower-layer 
transactions are tentatively committed that it can declare the higher-layer transaction a 
success. 
Based on that evaluation, it either COMMITs or ABORTs the higher-layer transaction. 
Assuming it decides to commit, it enters the second, commitment phase, which in the 
simplest case consists of simply changing its own state from PENDING to COMMITTED or 
ABORTED. If it is the highest-layer transaction, at that instant all of the lower-layer tenta­
tively committed transactions also become either COMMITTED or ABORTED. If it is itself 
nested in a still higher-layer transaction, it becomes tentatively committed and its com­
ponent transactions continue in the tentatively committed state also. We are 
implementing here a coordination protocol known as two-phase commit. When we 
implement multiple-site atomicity in the next section, the distinction between the two 
phases will take on additional clarity. 
Saltzer & Kaashoek Ch. 9, p. 84 
June 24, 2009 12:26 am

## Page 87

9.6 Atomicity across Layers and Multiple Sites 9–85  
If the system uses version histories for atomicity, the hierarchy of Figure 9.36 can be 
directly implemented by linking outcome records. If the system uses logs, a separate table 
of pending transactions can contain the hierarchy, and inquiries about the state of a 
transaction would involve examining this table. 
The concept of nesting transactions hierarchically is useful in its own right, but our 
particular interest in nesting is that it is the first of two building blocks for multiple-site 
transactions. To develop the second building block, we next explore what makes multi­
ple-site transactions different from single-site transactions. 
9.6.3 Multiple-Site Atomicity: Distributed Two-Phase Commit 
If a transaction requires executing component transactions at several sites that are sepa­
rated by a best-effort network, obtaining atomicity is more difficult because any of the 
messages used to coordinate the transactions of the various sites can be lost, delayed, or 
duplicated. In Chapter 4 we learned of a method, known as Remote Procedure Call 
(RPC) for performing an action at another site. In Chapter 7[on-line] we learned how 
to design protocols such as RPC with a persistent sender to ensure at-least-once execu­
tion and duplicate suppression to ensure at-most-once execution. Unfortunately, neither 
of these two assurances is exactly what is needed to ensure atomicity of a multiple-site 
transaction. However, by properly combining a two-phase commit protocol with persis­
tent senders, duplicate suppression, and single-site transactions, we can create a correct 
multiple-site transaction. We assume that each site, on its own, is capable of implement­
ing local transactions, using techniques such as version histories or logs and locks for all-
or-nothing atomicity and before-or-after atomicity. Correctness of the multiple-site ato­
micity protocol will be achieved if all the sites commit or if all the sites abort; we will have 
failed if some sites commit their part of a multiple-site transaction while others abort 
their part of that same transaction. 
Suppose the multiple-site transaction consists of a coordinator Alice requesting com­
ponent transactions X, Y, and Z of worker sites Bob, Charles, and Dawn, respectively. 
The simple expedient of issuing three remote procedure calls certainly does not produce 
a transaction for Alice because Bob may do X while Charles may report that he cannot 
do Y. Conceptually, the coordinator would like to send three messages, to the three 
workers, like this one to Bob: 
From: Alice  
To: Bob  
Re: my transaction 91  
if (Charles does Y and Dawn does Z) then do X, please. 
and let the three workers handle the details. We need some clue how Bob could accom­
plish this strange request. 
The clue comes from recognizing that the coordinator has created a higher-layer 
transaction and each of the workers is to perform a transaction that is nested in the 
higher-layer transaction. Thus, what we need is a distributed version of the two-phase 
commit protocol. The complication is that the coordinator and workers cannot reliably 
Saltzer & Kaashoek Ch. 9, p. 85 
June 24, 2009 12:26 am

## Page 88

9–86 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
communicate. The problem thus reduces to constructing a reliable distributed version of 
the two-phase commit protocol. We can do that by applying persistent senders and 
duplicate suppression. 
Phase one of the protocol starts with coordinator Alice creating a top-layer outcome 
record for the overall transaction. Then Alice begins persistently sending to Bob an RPC-
like message: 
From:Alice  
To: Bob  
Re: my transaction 271  
Please do X as part of my transaction. 
Similar messages go from Alice to Charles and Dawn, also referring to transaction 271, 
and requesting that they do Y and Z, respectively. As with an ordinary remote procedure 
call, if Alice doesn’t receive a response from one or more of the workers in a reasonable 
time she resends the message to the non-responding workers as many times as necessary 
to elicit a response. 
A worker site, upon receiving a request of this form, checks for duplicates and then 
creates a transaction of its own, but it makes the transaction a nested one, with its superior 
being Alice’s original transaction. It then goes about doing the pre-commit part of the 
requested action, reporting back to Alice that this much has gone well: 
From:Bob  
To: Alice  
Re: your transaction 271  
My part X is ready to commit. 
Alice, upon collecting a complete set of such responses then moves to the two-phase 
commit part of the transaction, by sending messages to each of Bob, Charles, and Dawn 
saying, e.g.: 
Two-phase-commit message #1: 
From:Alice  
To: Bob  
Re: my transaction 271  
PREPARE to commit X. 
Bob, upon receiving this message, commits—but only tentatively—or aborts. Having 
created durable tentative versions (or logged to journal storage its planned updates) and 
having recorded an outcome record saying that it is PREPARED either to commit or abort, 
Bob then persistently sends a response to Alice reporting his state: 
Saltzer & Kaashoek Ch. 9, p. 86 
June 24, 2009 12:26 am

## Page 89

9.6 Atomicity across Layers and Multiple Sites 9–87  
Two-phase-commit message #2: 
From:Bob 
To:Alice 
Re: your transaction 271 
I am PREPARED to commit my part. Have you decided to commit yet? Regards. 
or alternatively, a message reporting it has aborted. If Bob receives a duplicate request 
from Alice, his persistent sender sends back a duplicate of the PREPARED or ABORTED 
response. 
At this point Bob, being in the PREPARED state, is out on a limb. Just as in a local hier­
archical nesting, Bob must be able either to run to the end or to abort, to maintain that 
state of preparation indefinitely, and wait for someone else (Alice) to say which. In addi­
tion, the coordinator may independently crash or lose communication contact, 
increasing Bob’s uncertainty. If the coordinator goes down, all of the workers must wait 
until it recovers; in this protocol, the coordinator is a single point of failure. 
As coordinator, Alice collects the response messages from her several workers (perhaps 
re-requesting PREPARED responses several times from some worker sites). If all workers 
send PREPARED messages, phase one of the two-phase commit is complete. If any worker 
responds with an abort message, or doesn’t respond at all, Alice has the usual choice of 
aborting the entire transaction or perhaps trying a different worker site to carry out that 
component transaction. Phase two begins when Alice commits the entire transaction by 
marking her own outcome record COMMITTED. 
Once the higher-layer outcome record is marked as COMMITTED or ABORTED, Alice sends 
a completion message back to each of Bob, Charles, and Dawn: 
Two-phase-commit message #3 
From:Alice 
To:Bob 
Re: my transaction 271 
My transaction committed. Thanks for your help. 
Each worker site, upon receiving such a message, changes its state from PREPARED to COM­
MITTED, performs any needed post-commit actions, and exits. Meanwhile, Alice can go 
about other business, with one important requirement for the future: she must remem­
ber, reliably and for an indefinite time, the outcome of this transaction. The reason is 
that one or more of her completion messages may have been lost. Any worker sites that 
are in the PREPARED state are awaiting the completion message to tell them which way to 
go. If a completion message does not arrive in a reasonable period of time, the persistent 
sender at the worker site will resend its PREPARED message. Whenever Alice receives a 
duplicate PREPARED message, she simply sends back the current state of the outcome 
record for the named transaction. 
If a worker site that uses logs and locks crashes, the recovery procedure at that site has 
to take three extra steps. First, it must classify any PREPARED transaction as a tentative win­
ner that it should restore to the PREPARED state. Second, if the worker is using locks for 
Saltzer & Kaashoek Ch. 9, p. 87 
June 24, 2009 12:26 am

## Page 90

9–88 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
before-or-after atomicity, the recovery procedure must reacquire any locks the PREPARED 
transaction was holding at the time of the failure. Finally, the recovery procedure must 
restart the persistent sender, to learn the current status of the higher-layer transaction. If 
the worker site uses version histories, only the last step, restarting the persistent sender, 
is required. 
Since the workers act as persistent senders of their PREPARED messages, Alice can be 
confident that every worker will eventually learn that her transaction committed. But 
since the persistent senders of the workers are independent, Alice has no way of ensuring 
that they will act simultaneously. Instead, Alice can only be certain of eventual comple­
tion of her transaction. This distinction between simultaneous action and eventual 
action is critically important, as will soon be seen. 
If all goes well, two-phase commit of N worker sites will be accomplished in 3N mes­
sages, as shown in Figure 9.37: for each worker site a PREPARE message, a PREPARED message 
in response, and a COMMIT message. This 3N message protocol is complete and sufficient, 
although there are several variations one can propose. 
An example of a simplifying variation is that the initial RPC request and response 
could also carry the PREPARE and PREPARED messages, respectively. However, once a worker 
sends a PREPARED message, it loses the ability to unilaterally abort, and it must remain on 
the knife edge awaiting instructions from the coordinator. To minimize this wait, it is 
usually preferable to delay the PREPARE/PREPARED message pair until the coordinator knows 
that the other workers seem to be in a position to do their parts. 
Some versions of the distributed two-phase commit protocol have a fourth acknowl­
edgment message from the worker sites to the coordinator. The intent is to collect a 
complete set of acknowledgment messages—the coordinator persistently sends comple­
tion messages until every site acknowledges. Once all acknowledgments are in, the 
coordinator can then safely discard its outcome record, since every worker site is known 
to have gotten the word. 
A system that is concerned both about outcome record storage space and the cost of 
extra messages can use a further refinement, called presumed commit. Since one would 
expect that most transactions commit, we can use a slightly odd but very space-efficient 
representation for the value COMMITTED of an outcome record: non-existence. The coordi­
nator answers any inquiry about a non-existent outcome record by sending a COMMITTED 
response. If the coordinator uses this representation, it commits by destroying the out­
come record, so a fourth acknowledgment message from every worker is unnecessary. In 
return for this apparent magic reduction in both message count and space, we notice that 
outcome records for aborted transactions can not easily be discarded because if an 
inquiry arrives after discarding, the inquiry will receive the response COMMITTED. The coor­
dinator can, however, persistently ask for acknowledgment of aborted transactions, and 
discard the outcome record after all these acknowledgments are in. This protocol that 
leads to discarding an outcome record is identical to the protocol described in Chapter 
7[on-line] to close a stream and discard the record of that stream. 
Distributed two-phase commit does not solve all multiple-site atomicity problems. 
For example, if the coordinator site (in this case, Alice) is aboard a ship that sinks after 
Saltzer & Kaashoek Ch. 9, p. 88 
June 24, 2009 12:26 am

## Page 91

9.6 Atomicity across Layers and Multiple Sites 9–89 
Coordinator 
Worker 
Worker 
Worker 
Alice 
Bob 
Charles 
Dawn 
PREPARE X 
PREPARE Y 
PREPARE Z 
Bob is PREPARED to 
Charles is PREPARED to commit or abort 
Dawn is PREPARED to commit or abort 
COMMIT 
COMMIT 
COMMIT 
Time 
commit or abort 
log BEGIN 
log 
log BEGIN 
log PREPARED 
log COMMITTED 
COMMITTED 
FIGURE 9.37 
Timing diagram for distributed two-phase commit, using 3N messages. (The initial RPC 
request and response messages are not shown.) Each of the four participants maintains its 
own version history or recovery log. The diagram shows log entries made by the coordinator 
and by one of the workers. 
sending the PREPARE message but before sending the COMMIT or ABORT message the worker 
sites are in left in the PREPARED state with no way to proceed. Even without that concern, 
Alice and her co-workers are standing uncomfortably close to a multiple-site atomicity 
problem that, at least in principle, can not be solved. The only thing that rescues them is 
our observation that the several workers will do their parts eventually, not necessarily 
simultaneously. If she had required simultaneous action, Alice would have been in 
trouble. 
The unsolvable problem is known as the dilemma of the two generals. 
Saltzer & Kaashoek Ch. 9, p. 89 
June 24, 2009 12:26 am

## Page 92

9–90 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
9.6.4 The Dilemma of the Two Generals 
An important constraint on possible coordination protocols when communication is 
unreliable is captured in a vivid analogy, called the dilemma of the two generals.* Suppose 
that two small armies are encamped on two mountains outside a city. The city is well-
enough defended that it can repulse and destroy either one of the two armies. Only if the 
two armies attack simultaneously can they take the city. Thus the two generals who com­
mand the armies desire to coordinate their attack. 
The only method of communication between the two generals is to send runners 
from one camp to the other. But the defenders of the city have sentries posted in the val­
ley separating the two mountains, so there is a chance that the runner, trying to cross the 
valley, will instead fall into enemy hands, and be unable to deliver the message. 
Suppose that the first general sends this message: 
From:Julius Caesar  
To:Titus Labienus  
Date:11 January  
I propose to cross the Rubicon and attack at dawn tomorrow. OK? 
expecting that the second general will respond either with: 
From:Titus Labienus  
To:Julius Caesar;  
Date:11 January  
Yes, dawn on the 12th. 
or, possibly: 
From:Titus Labienus  
To:Julius Caesar  
Date:11 January  
No. I am awaiting reinforcements from Gaul. 
Suppose further that the first message does not make it through. In that case, the sec­
ond general does not march because no request to do so arrives. In addition, the first 
general does not march because no response returns, and all is well (except for the lost 
runner). 
Now, instead suppose the runner delivers the first message successfully and second 
general sends the reply “Yes,” but that the reply is lost. The first general cannot distin­
guish this case from the earlier case, so that army will not march. The second general has 
agreed to march, but knowing that the first general won’t march unless the “Yes” confir­
mation arrives, the second general will not march without being certain that the first 
* The origin of this analogy has been lost, but it was apparently ﬁrst described in print in 1977 by 
Jim N. Gray in his “Notes on Database Operating Systems”, reprinted in Operating Systems, Lecture 
Notes in Computer Science 60, Springer Verlag, 1978. At about the same time, Danny Cohen 
described another analogy he called the dating protocol, which is congruent with the dilemma of 
the two generals. 
Saltzer & Kaashoek Ch. 9, p. 90 
June 24, 2009 12:26 am

## Page 93

9.6 Atomicity across Layers and Multiple Sites 9–91  
general received the confirmation. This hesitation on the part of the second general sug­
gests that the first general should send back an acknowledgment of receipt of the 
confirmation: 
From:Julius Caesar  
To:Titus Labienus  
Date:11 January  
The die is cast. 
Unfortunately, that doesn’t help, since the runner carrying this acknowledgment may 
be lost and the second general, not receiving the acknowledgment, will still not march. 
Thus the dilemma. 
We can now leap directly to a conclusion: there is no protocol with a bounded num­
ber of messages that can convince both generals that it is safe to march. If there were such 
a protocol, the last message in any particular run of that protocol must be unnecessary to 
safe coordination because it might be lost, undetectably. Since the last message must be 
unnecessary, one could delete that message to produce another, shorter sequence of mes­
sages that must guarantee safe coordination. We can reapply the same reasoning 
repeatedly to the shorter message sequence to produce still shorter ones, and we conclude 
that if such a safe protocol exists it either generates message sequences of zero length or 
else of unbounded length. A zero-length protocol can’t communicate anything, and an 
unbounded protocol is of no use to the generals, who must choose a particular time to 
march. 
A practical general, presented with this dilemma by a mathematician in the field, 
would reassign the mathematician to a new job as a runner, and send a scout to check 
out the valley and report the probability that a successful transit can be accomplished 
within a specified time. Knowing that probability, the general would then send several 
(hopefully independent) runners, each carrying a copy of the message, choosing a num­
ber of runners large enough that the probability is negligible that all of them fail to 
deliver the message before the appointed time. (The loss of all the runners would be what 
Chapter 8[on-line] called an intolerable error.) Similarly, the second general sends many 
runners each carrying a copy of either the “Yes” or the “No” acknowledgment. This pro­
cedure provides a practical solution of the problem, so the dilemma is of no real 
consequence. Nevertheless, it is interesting to discover a problem that cannot, in princi­
ple, be solved with complete certainty. 
We can state the theoretical conclusion more generally and succinctly: if messages 
may be lost, no bounded protocol can guarantee with complete certainty that both gen­
erals know that they will both march at the same time. The best that they can do is accept 
some non-zero probability of failure equal to the probability of non-delivery of their last 
message. 
It is interesting to analyze just why we can’t we use a distributed two-phase commit 
protocol to resolve the dilemma of the two generals. As suggested at the outset, it has to 
do with a subtle difference in when things may, or must, happen. The two generals 
require, in order to vanquish the defenses of the city, that they march at the same time. 
Saltzer & Kaashoek Ch. 9, p. 91 
June 24, 2009 12:26 am

## Page 94

9–92 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
The persistent senders of the distributed two-phase commit protocol ensure that if the 
coordinator decides to commit, all of the workers will eventually also commit, but there 
is no assurance that they will do so at the same time. If one of the communication links 
goes down for a day, when it comes back up the worker at the other end of that link will 
then receive the notice to commit, but this action may occur a day later than the actions 
of its colleagues. Thus the problem solved by distributed two-phase commit is slightly 
relaxed when compared with the dilemma of the two generals. That relaxation doesn’t 
help the two generals, but the relaxation turns out to be just enough to allow us to devise 
a protocol that ensures correctness. 
By a similar line of reasoning, there is no way to ensure with complete certainty that 
actions will be taken simultaneously at two sites that communicate only via a best-effort 
network. Distributed two-phase commit can thus safely open a cash drawer of an ATM 
in Tokyo, with confidence that a computer in Munich will eventually update the balance 
of that account. But if, for some reason, it is necessary to open two cash drawers at dif­
ferent sites at the same time, the only solution is either the probabilistic approach or to 
somehow replace the best-effort network with a reliable one. The requirement for reli­
able communication is why real estate transactions and weddings (both of which are 
examples of two-phase commit protocols) usually occur with all of the parties in one 
room. 
9.7 A More Complete Model of Disk Failure (Advanced Topic) 
Section 9.2 of this chapter developed a failure analysis model for a calendar management 
program in which a system crash may corrupt at most one disk sector—the one, if any, 
that was being written at the instant of the crash. That section also developed a masking 
strategy for that problem, creating all-or-nothing disk storage. To keep that development 
simple, the strategy ignored decay events. This section revisits that model, considering 
how to also mask decay events. The result will be all-or-nothing durable storage, mean­
ing that it is both all-or-nothing in the event of a system crash and durable in the face of 
decay events. 
9.7.1 Storage that is Both All-or-Nothing and Durable 
In Chapter 8[on-line] we learned that to obtain durable storage we should write two 
or more replicas of each disk sector. In the current chapter we learned that to recover 
from a system crash while writing a disk sector we should never overwrite the previous 
version of that sector, we should write a new version in a different place. To obtain stor­
age that is both durable and all-or-nothing we combine these two observations: make 
more than one replica, and don’t overwrite the previous version. One easy way to do that 
would be to simply build the all-or-nothing storage layer of the current chapter on top 
of the durable storage layer of Chapter 8[on-line]. That method would certainly work 
but it is a bit heavy-handed: with a replication count of just two, it would lead to allo-
Saltzer & Kaashoek Ch. 9, p. 92 
June 24, 2009 12:26 am

## Page 95

9.7 A More Complete Model of Disk Failure (Advanced Topic) 9–93  
cating six disk sectors for each sector of real data. This is a case in which modularity has 
an excessive cost. 
Recall that the parameter that Chapter 8[on-line] used to determine frequency of 
checking the integrity of disk storage was the expected time to decay, Td. Suppose for the 
moment that the durability requirement can be achieved by maintaining only two cop­
ies. In that case, Td must be much greater than the time required to write two copies of 
a sector on two disks. Put another way, a large Td means that the short-term chance of a 
decay event is small enough that the designer may be able to safely neglect it. We can 
take advantage of this observation to devise a slightly risky but far more economical 
method of implementing storage that is both durable and all-or-nothing with just two 
replicas. The basic idea is that if we are confident that we have two good replicas of some 
piece of data for durability, it is safe (for all-or-nothing atomicity) to overwrite one of the 
two replicas; the second replica can be used as a backup to ensure all-or-nothing atom­
icity if the system should happen to crash while writing the first one. Once we are 
confident that the first replica has been correctly written with new data, we can safely 
overwrite the second one, to regain long-term durability. If the time to complete the two 
writes is short compared with Td, the probability that a decay event interferes with this 
algorithm will be negligible. Figure 9.38 shows the algorithm and the two replicas of the 
data, here named D0 and D1. 
An interesting point is that ALL_OR_NOTHING_DURABLE_GET does not bother to check the 
status returned upon reading D1—it just passes the status value along to its caller. The 
reason is that in the absence of decay CAREFUL_GET has no expected errors when reading 
data that CAREFUL_PUT was allowed to finish writing. Thus the returned status would be 
BAD only in two cases: 
1. CAREFUL_PUT of D1 was interrupted in mid-operation, or 
2. D1 was subject to an unexpected decay. 
The 
algorithm 
guarantees 
that 
the 
first 
case 
cannot 
happen. 
ALL_OR_NOTHING_DURABLE_PUT doesn’t begin CAREFUL_PUT on data D1 until after the comple­
tion of its CAREFUL_PUT on data D0. At most one of the two copies could be BAD because of 
a system crash during CAREFUL_PUT. Thus if the first copy (D0) is BAD, then we expect that 
the second one (D1) is OK. 
The risk of the second case is real, but we have assumed its probability to be small: it 
arises only if there is a random decay of D1 in a time much shorter than Td. In reading 
D1 we have an opportunity to detect that error through the status value, but we have no 
way to recover when both data copies are damaged, so this detectable error must be clas­
sified as untolerated. All we can do is pass a status report along to the application so that 
it knows that there was an untolerated error. 
There is one currently unnecessary step hidden in the SALVAGE program: if D0 is BAD, 
nothing is gained by copying D1 onto D0, since ALL_OR_NOTHING_DURABLE_PUT, which 
called SALVAGE, will immediately overwrite D0 with new data. The step is included 
because it allows SALVAGE to be used in a refinement of the algorithm. 
Saltzer & Kaashoek Ch. 9, p. 93 
June 24, 2009 12:26 am

## Page 96

9–94 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
In the absence of decay events, this algorithm would be just as good as the all-or-noth­
ing procedures of Figures 9.6 and 9.7, and it would perform somewhat better, since it 
involves only two copies. Assuming that errors are rare enough that recovery operations 
do not dominate performance, the usual cost of ALL_OR_NOTHING_DURABLE_GET is just one 
disk read, compared with three in the ALL_OR_NOTHING_GET algorithm. The cost of 
ALL_OR_NOTHING_DURABLE_PUT is two disk reads (in SALVAGE) and two disk writes, compared 
with three disk reads and three disk writes for the ALL_OR_NOTHING_PUT algorithm. 
That analysis is based on a decay-free system. To deal with decay events, thus making 
the scheme both all-or-nothing and durable, the designer adopts two ideas from the dis­
cussion of durability in Chapter 8[on-line], the second of which eats up some of the 
better performance: 
1.  Place the two copies, D0 and D1, in independent decay sets (for example write 
them on two different disk drives, preferably from different vendors). 
2.  Have a clerk run the SALVAGE program on every atomic sector at least once every 
Td seconds. 
1 procedure ALL_OR_NOTHING_DURABLE_GET (reference data, atomic_sector)  
2
 ds
 
← CAREFUL_GET (data, atomic_sector.D0)  
3 
if ds = BAD then  
4
 ds
 
← CAREFUL_GET (data, atomic_sector.D1)  
5 
return ds  
6 procedure ALL_OR_NOTHING_DURABLE_PUT (new_data, atomic_sector)  
7 
SALVAGE(atomic_sector)  
8
 ds
 
← CAREFUL_PUT (new_data, atomic_sector.D0)  
9
 ds
 
← CAREFUL_PUT (new_data, atomic_sector.D1)  
10 
return ds  
11 procedure SALVAGE(atomic_sector) 
//Run this program every Td seconds.  
12 
ds0 ← CAREFUL_GET (data0, atomic_sector.D0)  
13 
ds1 ← CAREFUL_GET (data1, atomic_sector.D1)  
14 
if ds0 = BAD then  
15 
CAREFUL_PUT (data1, atomic_sector.D0)  
16 
else if ds1 = BAD then  
17 
CAREFUL_PUT (data0, atomic_sector.D1)  
18 
if data0 ≠ data1 then  
19 
CAREFUL_PUT (data0, atomic_sector.D1)  
D0: 
FIGURE 9.38 
data0 
D1: 
data1 
Data arrangement and algorithms to implement all-or-nothing durable storage on top of the 
careful storage layer of Figure 8.12. 
Saltzer & Kaashoek Ch. 9, p. 94  
June 24, 2009 12:26 am

## Page 97

9.8 Case Studies: Machine Language Atomicity 9–95  
The clerk running the SALVAGE program performs 2N disk reads every Td seconds to 
maintain N durable sectors. This extra expense is the price of durability against disk 
decay. The performance cost of the clerk depends on the choice of Td, the value of N, 
and the priority of the clerk. Since the expected operational lifetime of a hard disk is usu­
ally several years, setting Td to a few weeks should make the chance of untolerated failure 
from decay negligible, especially if there is also an operating practice to routinely replace 
disks well before they reach their expected operational lifetime. A modern hard disk with 
a capacity of one terabyte would have about N = 109 kilobyte-sized sectors. If it takes 10 
milliseconds to read a sector, it would take about 2 x 107 seconds, or two days, for a clerk 
to read all of the contents of two one-terabyte hard disks. If the work of the clerk is sched­
uled to occur at night, or uses a priority system that runs the clerk when the system is 
otherwise not being used heavily, that reading can spread out over a few weeks and the 
performance impact can be minor. 
A few paragraphs back mentioned that there is the potential for a refinement: If we 
also run the SALVAGE program on every atomic sector immediately following every system 
crash, then it should not be necessary to do it at the beginning of every 
ALL_OR_NOTHING_DURABLE_PUT. That variation, which is more economical if crashes are 
infrequent and disks are not too large, is due to Butler Lampson and Howard Sturgis 
[Suggestions for Further Reading 1.8.7]. It raises one minor concern: it depends on the 
rarity of coincidence of two failures: the spontaneous decay of one data replica at about 
the same time that CAREFUL_PUT crashes in the middle of rewriting the other replica of that 
same sector. If we are convinced that such a coincidence is rare, we can declare it to be 
an untolerated error, and we have a self-consistent and more economical algorithm. 
With this scheme the cost of ALL_OR_NOTHING_DURABLE_PUT reduces to just two disk writes. 
9.8 Case Studies: Machine Language Atomicity 
9.8.1 Complex Instruction Sets:The General Electric 600 Line 
In the early days of mainframe computers, most manufacturers reveled in providing elab­
orate instruction sets, without paying much attention to questions of atomicity. The 
General Electric 600 line, which later evolved to be the Honeywell Information System, 
Inc., 68 series computer architecture, had a feature called “indirect and tally.” One could 
specify this feature by setting to ON a one-bit flag (the “tally” flag) stored in an unused 
high-order bit of any indirect address. The instruction 
Load register A from Y indirect. 
was interpreted to mean that the low-order bits of the cell with address Y contain another 
address, called an indirect address, and that indirect address should be used to retrieve 
the operand to be loaded into register A. In addition, if the tally flag in cell Y is ON, the 
processor is to increment the indirect address in Y by one and store the result back in Y. 
The idea is that the next time Y is used as an indirect address it will point to a different 
Saltzer & Kaashoek Ch. 9, p. 95 
June 24, 2009 12:26 am

## Page 98

9–96 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
operand—the one in the next sequential address in memory. Thus the indirect and tally 
feature could be used to sweep through a table. The feature seemed useful to the design­
ers, but it was actually only occasionally, because most applications were written in 
higher-level languages and compiler writers found it hard to exploit. On the other hand 
the feature gave no end of trouble when virtual memory was retrofitted to the product 
line. 
Suppose that virtual memory is in use, and that the indirect word is located in a page 
that is in primary memory, but the actual operand is in another page that has been 
removed to secondary memory. When the above instruction is executed, the processor 
will retrieve the indirect address in Y, increment it, and store the new value back in Y. 
Then it will attempt to retrieve the actual operand, at which time it discovers that it is 
not in primary memory, so it signals a missing-page exception. Since it has already mod­
ified the contents of Y (and by now Y may have been read by another processor or even 
removed from memory by the missing-page exception handler running on another pro­
cessor), it is not feasible to back out and act as if this instruction had never executed. The 
designer of the exception handler would like to be able to give the processor to another 
thread by calling a function such as AWAIT while waiting for the missing page to arrive. 
Indeed, processor reassignment may be the only way to assign a processor to retrieve the 
missing page. However, to reassign the processor it is necessary to save its current execu­
tion state. Unfortunately, its execution state is “half-way through the instruction last 
addressed by the program counter.” Saving this state and later restarting the processor in 
this state is challenging. The indirect and tally feature was just one of several sources of 
atomicity problems that cropped up when virtual memory was added to this processor. 
The virtual memory designers desperately wanted to be able to run other threads on 
the interrupted processor. To solve this problem, they extended the definition of the cur­
rent program state to contain not just the next-instruction counter and the program-
visible registers, but also the complete internal state description of the processor—a 216­
bit snapshot in the middle of the instruction. By later restoring the processor state to con­
tain the previously saved values of the next-instruction counter, the program-visible 
registers, and the 216-bit internal state snapshot, the processor could exactly continue 
from the point at which the missing-page alert occurred. This technique worked but it 
had two awkward side effects: 1) when a program (or programmer) inquires about the 
current state of an interrupted processor, the state description includes things not in the 
programmer’s interface; and 2) the system must be careful when restarting an interrupted 
program to make certain that the stored micro-state description is a valid one. If someone 
has altered the state description the processor could try to continue from a state it could 
never have gotten into by itself, which could lead to unplanned behavior, including fail­
ures of its memory protection features. 
9.8.2 More Elaborate Instruction Sets: The IBM System/370 
When IBM developed the System/370 by adding virtual memory to its System/360 
architecture, certain System/360 multi-operand character-editing instructions caused 
Saltzer & Kaashoek Ch. 9, p. 96 
June 24, 2009 12:26 am

## Page 99

9.8 Case Studies: Machine Language Atomicity 9–97  
atomicity problems. For example, the TRANSLATE instruction contains three arguments, 
two of which are addresses in memory (call them string and table) and the third of which, 
length, is an 8-bit count that the instruction interprets as the length of string. TRANSLATE 
takes one byte at a time from string, uses that byte as an offset in table, retrieves the byte 
at the offset, and replaces the byte in string with the byte it found in table. The designers 
had in mind that TRANSLATE could be used to convert a character string from one character 
set to another. 
The problem with adding virtual memory is that both string and table may be as long 
as 65,536 bytes, so either or both of those operands may cross not just one, but several 
page boundaries. Suppose just the first page of string is in physical memory. The TRANS­
LATE instruction works its way through the bytes at the beginning of string. When it 
comes to the end of that first page, it encounters a missing-page exception. At this point, 
the instruction cannot run to completion because data it requires is missing. It also can­
not back out and act as if it never started because it has modified data in memory by 
overwriting it. After the virtual memory manager retrieves the missing page, the problem 
is how to restart the half-completed instruction. If it restarts from the beginning, it will 
try to convert the already-converted characters, which would be a mistake. For correct 
operation, the instruction needs to continue from where it left off. 
Rather than tampering with the program state definition, the IBM processor design­
ers chose a dry run strategy in which the TRANSLATE instruction is executed using a hidden 
copy of the program-visible registers and making no changes in memory. If one of the 
operands causes a missing-page exception, the processor can act as if it never tried the 
instruction, since there is no program-visible evidence that it did. The stored program 
state shows only that the TRANSLATE instruction is about to be executed. After the proces­
sor retrieves the missing page, it restarts the interrupted thread by trying the TRANSLATE 
instruction from the beginning again, another dry run. If there are several missing pages, 
several dry runs may occur, each getting one more page into primary memory. When a 
dry run finally succeeds in completing, the processor runs the instruction once more, this 
time for real, using the program-visible registers and allowing memory to be updated. 
Since the System/370 (at the time this modification was made) was a single-processor 
architecture, there was no possibility that another processor might snatch a page away 
after the dry run but before the real execution of the instruction. This solution had the 
side effect of making life more difficult for a later designer with the task of adding mul­
tiple processors. 
9.8.3 The Apollo Desktop Computer and the Motorola M68000 Microprocessor 
When Apollo Computer designed a desktop computer using the Motorola 68000 micro­
processor, the designers, who wanted to add a virtual memory feature, discovered that 
the microprocessor instruction set interface was not atomic. Worse, because it was con­
structed entirely on a single chip it could not be modified to do a dry run (as in the IBM 
370) or to make it store the internal microprogram state (as in the General Electric 600 
line). So the Apollo designers used a different strategy: they installed not one, but two 
Saltzer & Kaashoek Ch. 9, p. 97 
June 24, 2009 12:26 am

## Page 100

9–98 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
Motorola 68000 processors. When the first one encounters a missing-page exception, it 
simply stops in its tracks, and waits for the operand to appear. The second Motorola 
68000 (whose program is carefully planned to reside entirely in primary memory) fetches 
the missing page and then restarts the first processor. 
Other designers working with the Motorola 68000 used a different, somewhat risky 
trick: modify all compilers and assemblers to generate only instructions that happen to 
be atomic. Motorola later produced a version of the 68000 in which all internal state reg­
isters of the microprocessor could be saved, the same method used in adding virtual 
memory to the General Electric 600 line. 
Exercises 
9.1   Locking up humanities: The registrar’s ofﬁce is upgrading its scheduling program for 
limited-enrollment humanities subjects. The plan is to make it multithreaded, but 
there is concern that having multiple threads trying to update the database at the 
same time could cause trouble. The program originally had just two operations: 
status ← REGISTER (subject_name)  
DROP (subject_name)  
where subject_name was a string such as “21W471”. The REGISTER procedure 
checked to see if there is any space left in the subject, and if there was, it 
incremented the class size by one and returned the status value ZERO. If there was no 
space, it did not change the class size; instead it returned the status value –1. (This 
is a primitive registration system—it just keeps counts!) 
As part of the upgrade, subject_name has been changed to a two-component 
structure: 
structure subject  
string subject_name  
lock slock  
and the registrar is now wondering where to apply the locking primitives, 
ACQUIRE (subject.slock)  
RELEASE (subject.slock)  
Here is a typical application program, which registers the caller for two humanities 
Saltzer & Kaashoek Ch. 9, p. 98  
June 24, 2009 12:26 am

## Page 101

Exercises 9–99  
subjects, hx and hy: 
procedure REGISTER_TWO (hx, hy)  
status ← REGISTER (hx)  
if status = 0 then  
status ← REGISTER (hy)  
if status = –1 then   
DROP (hx)  
return status; 
 9.1a.  The goal is that the entire procedure REGISTER_TWO should have the before-or-after 
property. Add calls for ACQUIRE and RELEASE to the REGISTER_TWO procedure that 
obey the simple locking protocol.
   9.1b.   Add calls to ACQUIRE and RELEASE that obey the two-phase locking protocol, and in 
addition postpone all ACQUIREs as late as possible and do all RELEASEs as early as 
possible. 
Louis Reasoner has come up with a suggestion that he thinks could simplify the job 
of programmers creating application programs such as REGISTER_TWO. His idea is to 
revise the two programs REGISTER and DROP by having them do the ACQUIRE and 
RELEASE internally. That is, the procedure: 
procedure REGISTER (subject)   
{ current code }  
return status 
would become instead: 
procedure REGISTER (subject)   
ACQUIRE (subject.slock)  
{ current code }  
RELEASE (subject.slock)  
return status  
9.1c. As usual, Louis has misunderstood some aspect of the problem. Give a brief 
explanation of what is wrong with this idea. 
1995–3–2a…c 
9.2 Ben and Alyssa are debating a ﬁne point regarding version history transaction 
disciplines and would appreciate your help. Ben says that under the mark point 
transaction discipline, every transaction should call MARK_POINT_ANNOUNCE as soon as 
possible, or else the discipline won't work. Alyssa claims that everything will come 
out correct even if no transaction calls MARK_POINT_ANNOUNCE. Who is right? 
2006-0-1 
9.3   Ben and Alyssa are debating another ﬁne point about the way that the version 
history transaction discipline bootstraps. The version of NEW_OUTCOME_RECORD given 
in the text uses TICKET as well as ACQUIRE and RELEASE. Alyssa says this is overkill—it 
Saltzer & Kaashoek Ch. 9, p. 99  
June 24, 2009 12:26 am

## Page 102

9–100 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
should be possible to correctly coordinate NEW_OUTCOME_RECORD using just ACQUIRE 
and RELEASE. Modify the pseudocode of Figure 9.30 to create a version of 
NEW_OUTCOME_RECORD that doesn't need the ticket primitive. 
9.4   You have been hired by Many-MIPS corporation to help design a new 32-register 
RISC processor that is to have six-way multiple instruction issue. Your job is to 
coordinate the interaction among the six arithmetic-logic units (ALUs) that will be 
running concurrently. Recalling the discussion of coordination, you realize that the 
ﬁrst thing you must do is decide what constitutes “correct” coordination for a 
multiple-instruction-issue system. Correct coordination for concurrent operations 
on a database was said to be: 
No matter in what order things are actually calculated, the ﬁnal result is always 
guaranteed to be one that could have been obtained by some sequential ordering of 
the concurrent operations. 
You have two goals: (1) maximum performance, and (2) not surprising a 
programmer who wrote a program expecting it to be executed on a single-
instruction-issue machine. 
Identify the best coordination correctness criterion for your problem. 
A.   Multiple instruction issue must be restricted to sequences of instructions that have 
non-overlapping register sets. 
B.   No matter in what order things are actually calculated, the ﬁnal result is always 
guaranteed to be one that could have been obtained by some sequential ordering of 
the instructions that were issued in parallel. 
C.   No matter in what order things are actually calculated, the ﬁnal result is always 
guaranteed to be the one that would have been obtained by the original ordering of 
the instructions that were issued in parallel. 
D.   The ﬁnal result must be obtained by carrying out the operations in the order 
speciﬁed by the original program. 
E.   No matter in what order things are actually calculated, the ﬁnal result is always 
guaranteed to be one that could have been obtained by some set of instructions 
carried out sequentially. 
F.  The six ALUs do not require any coordination. 
1997–0–02 
9.5   In 1968, IBM introduced the Information Management System (IMS) and it soon 
became one of the most widely used database management systems in the world. In 
fact, IMS is still in use today. At the time of introduction IMS used a before-or-after 
atomicity protocol consisting of the following two rules: 
•   A transaction may read only data that has been written by previously committed 
transactions. 
•   A transaction must acquire a lock for every data item that it will write. 
Saltzer & Kaashoek Ch. 9, p. 100  
June 24, 2009 12:26 am

## Page 103

Exercises 9–101  
Consider the following two transactions, which, for the interleaving shown, both 
adhere to the protocol: 
1 
BEGIN (t1); 
BEGIN (t2) 
2 
ACQUIRE (y.lock) 
3 
temp1 ← x 
4 
ACQUIRE (x.lock) 
5 
temp2 ← y 
6 
x ← temp2 
7 
y ← temp1 
8 
COMMIT (t1) 
9 
COMMIT (t2) 
Previously committed transactions had set x ← 3 and y ← 4. 
9.5a. After both transactions complete, what are the values of x and y? In what sense is 
this answer wrong? 
1982–3–3a 
9.5b.  In the mid-1970’s, this ﬂaw was noticed, and the before-or-after atomicity protocol 
was replaced with a better one, despite a lack of complaints from customers. Explain 
why customers may not have complained about the ﬂaw. 
1982–3–3b 
9.6   A system that attempts to make actions all-or-nothing writes the following type of 
records to a log maintained on non-volatile storage: 
• <STARTED i>  
action i starts. 
• <i, x, old, new> 
action i writes the value new over the value old 
for the variable x. 
• <COMMITTED i> 
action i commits. 
• <ABORTED i> 
 action
 
i aborts. 
• <CHECKPOINT i, j,…> 
At this checkpoint, actions i, j,… are pending. 
Actions start in numerical order. A crash occurs, and the recovery procedure ﬁnds 
Saltzer & Kaashoek Ch. 9, p. 101  
June 24, 2009 12:26 am

## Page 104

9–102 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After 
the following log records starting with the last checkpoint: 
<CHECKPOINT 17, 51, 52>  
<STARTED 53>  
<STARTED 54>  
<53, y, 5, 6>  
<53, x, 5, 9>  
<COMMITTED 53>  
<54, y, 6, 4>  
<STARTED 55>  
<55, z, 3, 4>  
<ABORTED 17>  
<51, q, 1, 9>  
<STARTED 56>  
<55, y, 4, 3>  
<COMMITTED 54>  
<55, y, 3, 7>  
<COMMITTED 51>  
<STARTED 57>  
<56, x, 9, 2>  
<56, w, 0, 1>  
<COMMITTED 56>  
<57, u, 2, 1>  
****************** crash happened here **************  
9.6a. Assume that the system is using a rollback recovery procedure. How much farther 
back in the log should the recovery procedure scan? 
9.6b.  Assume that the system is using a roll-forward recovery procedure. How much 
farther back in the log should the recovery procedure scan? 
9.6c. Which operations mentioned in this part of the log are winners and which are 
losers?
 9.6d.  What are the values of x and y immediately after the recovery procedure ﬁnishes? 
Why? 
1994–3–3 
9.7   The log of exercise 9.6 contains (perhaps ambiguous) evidence that someone didn’t 
follow coordination rules. What is that evidence? 
1994–3–4 
9.8   Roll-forward recovery requires writing the commit (or abort) record to the log 
before doing any installs to cell storage. Identify the best reason for this requirement. 
A. So that the recovery manager will know what to undo. 
B. So that the recovery manager will know what to redo. 
C.   Because the log is less likely to fail than the cell storage. 
D.   To minimize the number of disk seeks required. 
1994–3–5 
Saltzer & Kaashoek Ch. 9, p. 102  
June 24, 2009 12:26 am

## Page 105

Exercises 9–103  
9.9   Two-phase locking within transactions ensures that 
A. No deadlocks will occur. 
B. Results will correspond to some serial execution of the transactions. 
C.   Resources will be locked for the minimum possible interval. 
D.   Neither gas nor liquid will escape. 
E. Transactions will succeed even if one lock attempt fails. 
1997–3–03 
9.10   Pat, Diane, and Quincy are having trouble using e-mail to schedule meetings. Pat 
suggests that they take inspiration from the 2-phase commit protocol. 
9.10a. Which of the following protocols most closely resembles 2-phase commit? 
I. a. Pat requests everyone’s schedule openings. 
b. Everyone replies with a list but does not guarantee to hold all the times available. 
c. Pat inspects the lists and looks for an open time.  
If there is a time,  
Pat chooses a meeting time and sends it to everyone.  
Otherwise  
Pat sends a message canceling the meeting.  
II. a–c, as in protocol I. 
d. Everyone, if they received the second message,  
acknowledge receipt.  
Otherwise  
send a message to Pat asking what happened.  
III a–c, as in protocol I. 
d. Everyone, if their calendar is still open at the chosen time  
Send Pat an acknowledgment.  
Otherwise  
Send Pat apologies.  
e. Pat collects the acknowledgments. If all are positive  
Send a message to everyone saying the meeting is ON.  
Otherwise  
Send a message to everyone saying the meeting is OFF.  
f. Everyone, if they received the ON/OFF message,  
acknowledge receipt.  
Otherwise  
send a message to Pat asking what happened.  
IV. a–f, as in protocol III. 
g. Pat sends a message telling everyone that everyone has confirmed. 
h. Everyone acknowledges the confirmation.
   9.10b.  For the protocol you selected, which step commits the meeting time? 
1994–3–7 
Saltzer & Kaashoek Ch. 9, p. 103  
June 24, 2009 12:26 am

## Page 106

9–104 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
9.11   Alyssa P. Hacker needs a transaction processing system for updating information 
about her collection of 97 cockroaches.* 
9.11a. In her ﬁrst design, Alyssa stores the database on disk. When a transaction commits, 
it simply goes to the disk and writes its changes in place over the old data. What are 
the major problems with Alyssa’s system?
   9.11b.  In Alyssa’s second design, the only structure she keeps on disk is a log, with a 
reference copy of all data in volatile RAM. The log records every change made to 
the database, along with the transaction which the change was a part of. Commit 
records, also stored in the log, indicate when a transaction commits. When the 
system crashes and recovers, it replays the log, redoing each committed transaction, 
to reconstruct the reference copy in RAM. What are the disadvantages of Alyssa’s 
second design? 
To speed things up, Alyssa makes an occasional checkpoint of her database. To 
checkpoint, Alyssa just writes the entire state of the database into the log. When the 
system crashes, she starts from the last checkpointed state, and then redoes or 
undoes some transactions to restore her database. Now consider the ﬁve 
transactions in the illustration: 
T1 
T2 
T3 
T4 
T5 
begin 
commit 
time
crash
checkpoint 
Transactions T2, T3, and T5 committed before the crash, but T1 and T4 were still 
pending. 
9.11c. When the system recovers, after the checkpointed state is loaded, some 
transactions will need to be undone or redone using the log. For each transaction, 
* Credit for developing exercise 9.11 goes to Eddie Kohler. 
Saltzer & Kaashoek Ch. 9, p. 104  
June 24, 2009 12:26 am

## Page 107

Exercises 9–105 
mark off in the table whether that transaction needs to be undone, redone, or 
neither. 
Undone 
Redone 
Neither 
T1 
T2 
T3 
T4 
T5 
9.11d. Now, assume that transactions T2 and T3 were actually nested transactions: T2 was 
nested in T1, and T3 was nested in T2. Again, ﬁll in the table 
Undone 
Redone 
Neither 
T1 
T2 
T3 
T4 
T5 
1996–3–3 
9.12 Alice is acting as the coordinator for Bob and Charles in a two-phase commit 
protocol. Here is a log of the messages that pass among them: 
1 
Alice ⇒ Bob: 
please do X 
2 
Alice ⇒ Charles: please do Y 
3 
Bob ⇒ Alice: 
done with X 
4 
Charles ⇒ Alice: done with Y 
5 
Alice ⇒ Bob: 
PREPARE to commit or abort 
6 
Alice ⇒ Charles: PREPARE to commit or abort 
7 
Bob ⇒ Alice: 
PREPARED 
8 
Charles ⇒ Alice: PREPARED 
9 
Alice ⇒ Bob: 
COMMIT 
10 
Alice ⇒ Charles: COMMIT 
At which points in this sequence is it OK for Bob to abort his part of the 
Saltzer & Kaashoek Ch. 9, p. 105 
June 24, 2009 12:26 am

## Page 108

9–106 
CHAPTER 9 Atomicity: All-or-Nothing and Before-or-After  
transaction? 
A. After Bob receives message 1 but before he sends message 3. 
B. After Bob sends message 3 but before he receives message 5. 
C. After Bob receives message 5 but before he sends message 7. 
D. After Bob sends message 7 but before he receives message 9. 
E. After Bob receives message 9. 
2008–3–11 
Additional exercises relating to Chapter 9 can be found in problem sets 29 
through 40. 
Saltzer & Kaashoek Ch. 9, p. 106 
June 24, 2009 12:26 am

## Page 109

CHAPTER
Glossary for Chapter 9  
abort—Upon deciding that an all-or-nothing action cannot or should not commit, to 
undo all of the changes previously made by that all-or-nothing action. After aborting, 
the state of the system, as viewed by anyone above the layer that implements the all-or­
nothing action, is as if the all-or-nothing action never existed. Compare with commit. 
[Ch. 9] 
all-or-nothing atomicity—A property of a multistep action that if an anticipated failure 
occurs during the steps of the action, the effect of the action from the point of view of 
its invoker is either never to have started or else to have been accomplished completely. 
Compare with before-or-after atomicity and atomic. [Ch. 9] 
archive—A record, usually kept in the form of a log, of old data values, for auditing, 
recovery from application mistakes, or historical interest. [Ch. 9] 
atomic (adj.); atomicity (n.)—A property of a multistep action that there be no evidence 
that it is composite above the layer that implements it. An atomic action can be before-
or-after, which means that its effect is as if it occurred either completely before or 
completely after any other before-or-after action. An atomic action can also be all-or­
nothing, which means that if an anticipated failure occurs during the action, the effect 
of the action as seen by higher layers is either never to have started or else to have 
completed successfully. An atomic action that is both all-or-nothing and before-or-after 
is known as a transaction. [Ch. 9] 
atomic storage—Cell storage for which a multicell PUT can have only two possible 
outcomes: (1) it stores all data successfully, or (2) it does not change the previous data at 
all. In consequence, either a concurrent thread or (following a failure) a later thread 
doing a GET will always read either all old data or all new data. Computer architectures 
in which multicell PUTs are not atomic are said to be subject to write tearing. [Ch. 9] 
before-or-after atomicity—A property of concurrent actions: Concurrent actions are 
before-or-after actions if their effect from the point of view of their invokers is the same 
as if the actions occurred either completely before or completely after one another. One 
consequence is that concurrent before-or-after software actions cannot discover the 
composite nature of one another (that is, one action cannot tell that another has multiple 
steps). A consequence in the case of hardware is that concurrent before-or-after WRITEs to 
the same memory cell will be performed in some order, so there is no danger that the cell 
will end up containing, for example, the OR of several WRITE values. The database 
literature uses the words “isolation” and “serializable”, the operating system literature 
uses the words “mutual exclusion” and “critical section”, and the computer architecture 
literature uses the unqualiﬁed word “atomicity” for this concept. Compare with all-or­
nothing atomicity and atomic. [Ch. 9] 
blind write—An update to a data value X by a transaction that did not previously read X. 
[Ch. 9] 
9–107  
Saltzer & Kaashoek Ch. 9, p. 107 
June 24, 2009 12:26 am

## Page 110

9–108 
Glossary for Chapter 9 
cell storage—Storage in which a WRITE or PUT operates by overwriting, thus destroying 
previously stored information. Many physical storage devices, including magnetic disk 
and CMOS random access memory, implement cell storage. Compare with journal 
storage. [Ch. 9] 
checkpoint—1. (n.) Information written to non-volatile storage that is intended to speed 
up recovery from a crash. 2 (v.) To write a checkpoint. [Ch. 9] 
close-to-open consistency—A consistency model for ﬁle operations. When a thread opens 
a ﬁle and performs several write operations, all of the modiﬁcations weill be visible to 
concurrent threads only after the ﬁrst thread closes the ﬁle.  [Ch. 4] 
coheerence—See read/write coherence or cache coherence. 
commit—To renounce the ability to abandon an all-or-nothing action unilaterally. One 
usually commits an all-or-nothing action before making its results available to 
concurrent or later all-or-nothing actions. Before committing, the all-or-nothing action 
can be abandoned and one can pretend that it had never been undertaken. After 
committing, the all-or-nothing action must be able to complete. A committed all-or­
nothing action cannot be abandoned; if it can be determined precisely how far its results 
have propagated, it may be possible to reverse some or all of its effects by compensation. 
Commitment also usually includes an expectation that the results preserve any 
appropriate invariants and will be durable to the extent that the application requires 
those properties. Compare with compensate and abort. [Ch. 9] 
compensate (adj.); compensation (n.)—To perform an action that reverses the effect of 
some previously committed action. Compensation is intrinsically application 
dependent; it is easier to reverse an incorrect accounting entry than it is to undrill an 
unwanted hole. [Ch. 9] 
do action—(n.) Term used in some systems for a redo action. [Ch. 9] 
exponential random backoff—A form of exponential backoff in which an action that 
repeatedly encounters interference repeatedly doubles (or, more generally, multiplies by 
a constant greater than one) the size of an interval from which it randomly chooses its 
next delay before retrying. The intent is that by randomly changing the timing relative 
to other, interfering actions, the interference will not recur. [Ch. 9] 
force—(v.) When output may be buffered, to ensure that a previous output value has 
actually been written to durable storage or sent as a message. Caches that are not write-
through usually have a feature that allows the invoker to force some or all of their 
contents to the secondary storage medium. [Ch. 9] 
install—In a system that uses logs to achieve all-or-nothing atomicity, to write data to cell 
storage. [Ch. 9] 
journal storage—Storage in which a WRITE or PUT appends a new value, rather than 
overwriting a previously stored value. Compare with cell storage. [Ch. 9] 
lock point—In a system that provides before-or-after atomicity by locking, the ﬁrst instant 
in a before-or-after action when every lock that will ever be in its lock set has been 
Saltzer & Kaashoek Ch. 9, p. 108 
June 24, 2009 12:26 am

## Page 111

Glossary for Chapter 9 9–109  
acquired. [Ch. 9] 
lock set—The collection of all locks acquired during the execution of a before-or-after 
action. [Ch. 9] 
log—1. (n.) A specialized use of journal storage to maintain an append-only record of some 
application activity. Logs are used to implement all-or-nothing actions, for performance 
enhancement, for archiving, and for reconciliation. 2. (v.) To append a record to a log. 
[Ch. 9] 
logical locking—Locking of higher-layer data objects such as records or ﬁelds of a database. 
Compare with physical locking. [Ch. 9] 
mark point—1. (adj.) An atomicity-assuring discipline in which each newly created action 
n must wait to begin reading shared data objects until action (n – 1) has marked all of 
the variables it intends to modify. 2. (n.) The instant at which an action has marked all 
of the variables it intends to modify. [Ch. 9] 
optimistic concurrency control—A concurrency control scheme that allows concurrent 
threads to proceed even though a risk exists that they will interfere with each other, with 
the plan of detecting whether there actually is interference and, if necessary, forcing one 
of the threads to abort and retry. Optimistic concurrency control is an effective 
technique in situations where interference is possible but not likely. Compare with 
pessimistic concurrency control. [Ch. 9] 
page fault—See missing-page exception. 
pair-and-spare—See pair-and-compare. 
pending—A state of an all-or-nothing action, when that action has not yet either 
committed or aborted. Also used to describe the value of a variable that was set or 
changed by a still-pending all-or-nothing action. [Ch. 9] 
pessimistic concurrency control—A concurrency control scheme that forces a thread to 
wait if there is any chance that by proceeding it may interfere with another, concurrent, 
thread. Pessimistic concurrency control is an effective technique in situations where 
interference between concurrent threads has a high probability. Compare with optimistic 
concurrency control. [Ch. 9] 
physical locking—Locking of lower-layer data objects, typically chunks of data whose 
extent is determined by the physical layout of a storage medium. Examples of such 
chunks are disk sectors or even an entire disk. Compare with logical locking. [Ch. 9] 
prepaging—An optimization for a multilevel memory manager in which the manager 
predicts which pages might be  needed and brings them into the primary memory before 
the application demands them. Compare with demand algorithm. 
prepared—In a layered or multiple-site all-or-nothing action, a state of a component action 
that has announced that it can, on command, either commit or abort. Having reached 
this state, it awaits a decision from the higher-layer coordinator of the action. [Ch. 9] 
presented load—See offered load. 
Saltzer & Kaashoek Ch. 9, p. 109 
June 24, 2009 12:26 am

## Page 112

9–110 
Glossary for Chapter 9 
progress—A desirable guarantee provided by an atomicity-assuring mechanism: that 
despite potential interference from concurrency some useful work will be done. An 
example of such a guarantee is that the atomicity-assuring mechanism will not abort at 
least one member of the set of concurrent actions. In practice, lack of a progress 
guarantee can sometimes be repaired by using exponential random backoff. In formal 
analysis of systems, progress is one component of a property known as “liveness”. 
Progress is an assurance that the system will move toward some speciﬁed goal, whereas 
liveness is an assurance that the system will eventually reach that goal. [Ch. 9] 
redo action—An application-speciﬁed action that, when executed during failure recovery, 
produces the effect of some committed component action whose effect may have been 
lost in the failure. (Some systems call this a “do action”. Compare with undo action.) [Ch.
 9] 
roll-forward recovery—A write-ahead log protocol with the additional requirement that 
the application log its outcome record before it performs any install actions. If there is a 
failure before the all-or-nothing action passes its commit point, the recovery procedure 
does not need to undo anything; if there is a failure after commit, the recovery procedure 
can use the log record to ensure that cell storage installs are not lost. Also known as redo 
logging. Compare with rollback recovery. [Ch. 9] 
rollback recovery—A write-ahead log protocol with the additional requirement that the 
application perform all install actions before logging an outcome record. If there is a 
failure before the all-or-nothing action commits, a recovery procedure can use the log 
record to undo the partially completed all-or-nothing action. Also known as undo 
logging. Compare with roll-forward recovery. [Ch. 9] 
serializable—A property of before-or-after actions, that even if several operate 
concurrently, the result is the same as if they had acted one at a time, in some sequential 
(in other words, serial) order. [Ch. 9] 
shadow copy—A working copy of an object that an all-or-nothing action creates so that it 
can make several changes to the object while the original remains unmodiﬁed. When the 
all-or-nothing action has made all of the changes, it then carefully exchanges the working 
copy with the original, thus preserving the appearance that all of the changes occurred 
atomically. Depending on the implementation, either the original or the working copy 
may be identiﬁed as the “shadow” copy, but the technique is the same in either case. [Ch.
 9] 
simple locking—A locking protocol for creating before-or-after actions requiring that no 
data be read or written before reaching the lock point. For the atomic action to also be 
all-or-nothing, a further requirement is that no locks be released before commit (or 
abort). Compare with two-phase locking. [Ch. 9] 
simple serialization—An atomicity protocol requiring that each newly created atomic 
action must wait to begin execution until all previously started atomic actions are no 
longer pending. [Ch. 9] 
transaction—A multistep action that is both atomic in the face of failure and atomic in the 
Saltzer & Kaashoek Ch. 9, p. 110 
June 24, 2009 12:26 am

## Page 113

Glossary for Chapter 9 9–111 
face of concurrency. That is, it is both all-or-nothing and before-or-after. [Ch. 9] 
transactional memory—A memory model in which multiple references to primary 
memory are both all-or-nothing and before-or-after. [Ch. 9] 
two generals dilemma—An intrinsic problem that no ﬁnite protocol can guarantee to 
simultaneously coordinate state values at two places that are linked by an unreliable 
communication network.  [Ch. 9] 
two-phase commit—A protocol that creates a higher-layer transaction out of separate, 
lower-layer transactions. The protocol ﬁrst goes through a preparation (sometimes called 
voting) phase, at the end of which each lower-layer transaction reports either that it 
cannot perform its part or that it is prepared to either commit or abort. It then enters a 
commitment phase in which the higher-layer transaction, acting as a coordinator, makes 
a ﬁnal decision—thus the name two-phase. Two-phase commit has no connection with 
the similar-sounding term two-phase locking. [Ch. 9] 
two-phase locking—A locking protocol for before-or-after atomicity that requires that no 
locks be released until all locks have been acquired (that is, there must be a lock point). 
For the atomic action to also be all-or-nothing, a further requirement is that no locks for 
objects to be written be released until the action commits.  Compare with simple locking. 
Two-phase locking has no connection with the similar-sounding term two-phase commit. 
[Ch. 9] 
undo action—An application-speciﬁed action that, when executed during failure recovery 
or an abort procedure, reverses the effect of some previously performed, but not yet 
committed, component action. The goal is that neither the original action nor its reversal 
be visible above the layer that implements the action. Compare with redo and compensate. 
[Ch. 9] 
version history—The set of all values for an object or variable that have ever existed, stored 
in journal storage. [Ch. 9] 
write-ahead-log (WAL) protocol—A recovery protocol that requires appending a log 
record in journal storage before installing the corresponding data in cell storage. [Ch. 9] 
write tearing—See atomic storage. 
Saltzer & Kaashoek Ch. 9, p. 111 
June 24, 2009 12:26 am

## Page 114

9–112 
Glossary for Chapter 9  
Saltzer & Kaashoek Ch. 9, p. 112 
June 24, 2009 12:26 am

## Page 115

CHAPTER
Index of Chapter 9 
Design principles and hints appear in underlined italics. Procedure names appear in SMALL 
CAPS. Page numbers in bold face are in the chapter Glossary. 
A 
abort  9–27, 9–107 
ACQUIRE  9–70 
action  9–3 
adopt sweeping simplifications  9–3, 9–29, 
9–30, 9–47 
all-or-nothing atomicity  9–21, 9–107 
archive  9–37, 9–107 
log  9–40 
atomic  9–107 
action  9–3, 9–107 
storage  9–107 
atomicity  9–3, 9–19, 9–107 
all-or-nothing  9–21, 9–107 
before-or-after  9–54, 9–107 
log  9–40 
B 
backoff 
exponential random  9–78, 9–107, 
9–108 
before-or-after atomicity  9–54, 9–107 
blind write  9–49, 9–66, 9–107 
blocking read  9–11 
bootstrapping  9–21, 9–43, 9–61, 9–80 
C 
cell 
storage  9–31, 9–107, 9–108 
checkpoint  9–51, 9–107, 9–108 
close-to-open consistency  9–107, 9–108 
commit  9–27, 9–107, 9–108 
two-phase  9–84, 9–107, 9–111 
compensation  9–107, 9–108 
consistency 
close-to-open  9–107, 9–108 
external time  9–18  
sequential  9–18  
D 
deadlock  9–76 
dependent outcome record  9–81 
design principles 
adopt sweeping simplifications  9–3, 9–29, 
9–30, 9–47  
end-to-end argument  9–79  
golden rule of atomicity  9–26, 9–42  
law of diminishing returns  9–53  
dilemma of the two generals  9–90, 9–107, 
9–111 
diminishing returns, law of  9–53 
discipline 
simple locking  9–72, 9–107, 9–110 
systemwide locking  9–70 
two-phase locking  9–73, 9–107, 9–111 
do action (see redo action) 
dry run  9–97 
durability 
log  9–40 
E 
end-to-end argument  9–79 
exponential 
random backoff  9–78, 9–107, 9–108 
external time consistency  9–18 
F 
force  9–53, 9–107, 9–108 
G 
golden rule of atomicity  9–26, 9–42 
granularity  9–71 
9–113 
Saltzer & Kaashoek Ch. 9, p. 113 
June 24, 2009 12:26 am

## Page 116

Index of Chapter  
9–114  
H 
high-water mark  9–65 
hints 
optimize for the common case  9–39 
I 
idempotent  9–47 
IMS (see Information Management System) 
in-memory database  9–39 
Information Management System  9–100 
install  9–39, 9–107, 9–108 
J 
journal storage  9–31, 9–107, 9–108 
L 
law of diminishing returns  9–53 
livelock  9–78 
lock  9–69 
compatibility mode  9–76  
manager  9–70  
point  9–72, 9–107, 9–108  
set  9–72, 9–107, 9–109  
locking discipline 
simple  9–72, 9–107, 9–110 
systemwide  9–70 
two-phase  9–73, 9–107, 9–111 
log  9–39, 9–107, 9–109 
archive  9–40 
atomicity  9–40 
durability  9–40 
performance  9–40 
record  9–42 
redo  9–50, 9–107, 9–110 
sequence number  9–53 
undo  9–50, 9–107, 9–110 
write-ahead  9–42, 9–107, 9–111 
logical 
locking  9–75, 9–107, 9–109 
M 
mark point  9–58, 9–107, 9–109 
memory 
transactional  9–69, 9–107, 9–111 
multiple 
-reader, single-writer protocol  9–76 
N 
nested outcome record  9–86 
non-blocking read  9–12 
O 
optimistic concurrency control  9–63, 
9–107, 9–109 
optimize for the common case  9–45 
optimize for the common case  9–39 
outcome record  9–32 
P 
pending  9–32, 9–107, 9–109 
performance log  9–40 
pessimistic concurrency control  9–63, 
9–107, 9–109 
physical 
locking  9–75, 9–107, 9–109 
prepaging  9–107, 9–109 
PREPARED 
message  9–87 
state  9–107, 9–109 
presumed commit  9–88 
progress  9–77, 9–107, 9–110 
protocol 
two-phase commit  9–84, 9–107, 9–111 
R 
random 
backoff, exponential  9–78, 9–107, 
9–108 
read-capture  9–63 
redo 
action  9–43, 9–107, 9–110 
log  9–50, 9–107, 9–110 
register renaming  9–67 
RELEASE  9–70 
reorder buffer  9–67 
representations 
version history  9–55 
roll-forward recovery  9–50, 9–107, 9–110 
Saltzer & Kaashoek Ch. 9, p. 114 
June 24, 2009 12:26 am

## Page 117

Index of Chapter  
9–115  
rollback recovery  9–50, 9–107, 9–110 
S 
sequence coordination  9–13 
sequential consistency  9–18 
serializable  9–18, 9–107, 9–110 
shadow copy  9–29, 9–107, 9–110 
simple 
locking discipline  9–72, 9–107, 9–110 
serialization  9–54, 9–107, 9–110 
single-writer, multiple-reader protocol  9–76 
snapshot isolation  9–68 
storage 
atomic  9–107 
cell  9–31, 9–107, 9–108 
journal  9–31, 9–107, 9–108 
sweeping simplifications 
(see adopt sweeping simplifications) 
systemwide lock  9–70 
T 
tentatively committed  9–82 
transaction  9–3, 9–4, 9–107, 9–110 
transactional memory  9–69, 9–107, 9–111 
TRANSFER operation  9–5 
two generals dilemma  9–90, 9–107, 9–111 
two-phase 
commit  9–84, 9–107, 9–111 
locking discipline  9–73, 9–107, 9–111 
U 
undo 
action  9–43, 9–107, 9–111 
log  9–50, 9–107, 9–110 
V 
version history  9–30, 9–107, 9–111 
W 
WAL (see write-ahead log) 
write-ahead log  9–42, 9–107, 9–111 
write tearing  9–107 
Saltzer & Kaashoek Ch. 9, p. 115 
June 24, 2009 12:26 am

## Page 118

Index of Chapter  
9–116  
Saltzer & Kaashoek Ch. 9, p. 116 
June 24, 2009 12:26 am