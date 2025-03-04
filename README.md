//////////////////////////////////////////////////////////////////////////////
// "NetPIPE" -- Network Protocol Independent Performance Evaluator.
// NetPIPE is free software distrubuted under the GPL license.
//
// This code was originally developed ~1998 by Quinn Snell, Armin Mikler,
// John Gustafson, and Guy Helmer at Ames Laboratory. (core, MPI, TCP)
//
// Further developed by Dave Turner at Ames Lab 2000-2005 with many contributions
// from ISU students (Bogdan Vasiliu, Adam Oline, Xuehua Chen, Brian Smith).
// (many of the low level modules, complete rewrite of the core)
//
// Currently being developed Fall 2014- by Dave Turner at Kansas State University
// (io, aggregate, ibverbs)
// Email: DrDaveTurner@gmail.com
//////////////////////////////////////////////////////////////////////////////

    NetPIPE is a communication benchmark for testing networks, memory movement,
and IO subsystems.  Network tests involve the typical ping-pong of messages
across the message-passing or native communication layer, with support for
MPI, InfiniBand verbs, TCP, and shmem.  The ability to test at multiple 
communication layers makes it useful in determining where performance 
problems are occuring in a network.
     NetPIPE offers more rigorous testing than other benchmarks that often 
only test message sizes that are factors of 2.  This has been instrumental 
in detecting drop outs that would otherwise go unnoticed.  There are also
a variety of individual tests  and test parameters that can be used to
more fully evaluate a network.  These include full integrity testing,
testing with and without memory cache effects, streaming options to 
measure performance in one direction only, and fine tuning of all testing
parameters.
    Aggregate bandwidth can also be tested by measuring the performance 
between multiple pairs of processes on different nodes.  This allows for
measurement of the maximum bandwdith that you can get in and out of a node,
the maximum traffic that can be put across a link in a supercomputer, or
the saturation level for a network switch.  This ability most closely
simulates the network load that applications will put on an HPC system.
It has been used to isolate problems with some of the new 100 Gbps networks
that require careful tuning to optimize.
    Memory movement rates can be measured using the memcpy module with
testing both with and without cache effects.  IO rates can be measured
for the common C functions as well as file cp and scp rates.
    The NetPIPE output files provide the average bandwidth measured for 
each message size, plus the minimum, maximum, and average transfer time 
for each datapoint.  The geplot script can provide a quick graph using 
gnuplot and your favorite viewer (Preview under OS-X).  


Compiling NetPIPE
=================

Compile using make and the module name {mpi|tcp|ibverbs|memcpy|disk|theo}
Run using mpirun
output goes to np.out or the file specified with '-o np.output.filename'

General Options - for the NetPIPE core
===============

mpirun NPmpi --usage         this will list all options for the core and the module

      --bidir       Send data in both directions at the same time
      --burst       Burst pre-post all receives before timing (vendor cheat)
      --nocache     Invalidate cache (data comes from main memory)
      --integrity   Message integrity check instead of performance measurement
      --start #     Set the lower bound for the message sizes to test
      --end #       Set the upper bound for the message sizes to test
      -o filename   Set the output file name (default np.out)
      --offset #    Offset the send and recv buffers from page alignment
      --pert #      Set the perturbation size (default +/- 3 bytes)
      --stream      Stream data in one direction only
      --repeats #   Set a constant number of repeats per trial
      --quick       quicker run with no perturbations and lower time per trial
      -Q            Quick and dirty test of 1 repeat per trial
      --printhostnames   Each proc will print its hostname


MPI module    // examples for 2 procs on different hosts, setup in an OpenMPI hostfile
==========

make mpi

  // Test run for 2 processes on different hosts, unidirectional then bi-directional

cat mpihosts.2
  host1 slots=1
  host2 slots=1

mpirun -np 2 --hostfile mpihosts.2 NPmpi -o np.mpi
mpirun -np 2 --hostfile mpihosts.2 NPmpi --bidir --async -o np.mpi.bidirectional

  // Aggregate bandwidth measured using 32 processes on a host each paired with
  //    1 of 32 processes on the second host

cat mpihosts.all
  host1 slots=32
  host2 slots=32

mpirun -np 64 --hostfile mpihosts.all NPmpi --bidir --async -o np.mpi.aggregate

  --async        use asynchronous MPI_Irecv() to pre-post
  --sync         use synchronous  MPI_SSend() instead of blocking
  --anysource    receive using the MPI_ANY_SOURCE flag

    The aggregate tests can also be used to measure the maximum bandwidth
that a link in a supercomputer can handle when it's greater than what each
node can deliver.  Running on 8 nodes in a given row for example would 
pair up nodes 0-7, 1-6, 2-5, 3-4 to overload the link between 3 & 4.
     They can also be used to saturate a network switch.
For a 42 port OmniPath switch I used 42 nodes with 28 processes / node
to test for any blocking on the switch (none found).


InfiniBand Verbs module    // (Work still in progress, RC mostly done, UD not
=======================

    I still have a few issues with the RC module as it freezes up for message
sizes around 64 kB.  The UD side is programmed up but is still a work in progress.

make ibverbs
mpirun -np 2 --hostfile mpihosts.2 NPibverbs --device mlx4_0 --port 1

  --mtu #        mtu size of 256, 512, 1024, 2048, 4096
  --polling      Poll for receive completion
  --events       Use event notification for receive completion
    --acks #     For events, number of acks to do at once (default 1)
  --ud           Use Unreliable Datagrams instead of a Reliable Connection
    --sendqueues #   Number of send QP's to strip UD MTU's accross (default 1)
    --memcpy     For UD, do an extra memory copy (default)
  --rdma         use 1-sided RDMA puts
  --inline       Pass small messages inline
  --pinned       (default)  or  --unpinned
  --device name  (default is the first device found)
  --port #       (default is the first active port on the device)


TCP module
==========

make tcp
mpirun -np 2 --hostfile mpihosts.2 NPtcp --bidir

  --reset          reset the socket between trials
  --buffersize #   try to set the TCP send and recv socket buffer sizes


memcpy module    (This module can do multiple concurrent memory copies)
=============

    Memory movement can be measured with cache effects or without cache
effects where the data being copied comes from a different memory location
each time.  This allows for a more complete analysis of the memory movement
capabilities of a system than the Stream benchmark for example where 
it just measures for a single very large memory transfer, large enough to
avoid cache effects.
    You can also use multiple MPI tasks to measure the aggregate memory
access capabilites.

make memcpy
mpirun -np 1  NPmemcpy --nocache
mpirun -np 16 NPmemcpy --cache

      --memcpy      uses asm opt memcpy() (default)
      --memset      uses asm opt memset()
      --dcopy       unrolled copy  by doubles loop
      --dread       unrolled read  by doubles loop
      --dwrite      unrolled write by doubles loop


disk module    (module using C IO function calls)
===========

    Measure the input and output rates from C functions like
putc/getc, fprintf/fscanf, fwrite/fread, as well as cp/scp commands.
Binary streaming output is limited by the disk access rates, while
most other functions are limited by both the size of the data being
read or written and the cost of translation to or from ascii.
    In order to get accurate results for input tests you must take
care to purge the IO buffer.  Either create the files on one node
and do the read tests from a second node, or you can create or move
a file larger than the node's memory size to purge the memory buffer.

make disk
mpirun -np 1 NPdisk --write --string 100 --datafile np.string100
mpirun -np 1 NPdisk --read  --binary --datafile /scratch/user/np.binary

  NPdisk specific input arguments (default is to write chars to datafile)
  --datafile filename    Use 'filename' to read or write test data
  --read                 Test reading from datafile
  --write                Test writing to datafile
  --char                 Read/write ascii char to/from the datafile
  --string #             Read/write string of # chars to/from the datafile
  --double               Read/write formatted doubles to datafile
  --unformatted          Stream unformatted doubles to/from the datafile
  --binary               Stream the binary data in/out in 1 chunk
  --flush                fflush() data after each write

  NPdisk arguments for copying files
  --cp dirname           Files will be copied to this directory
  --scp dirname          Files will be copied to this user@ssh_host:directory
                         Must be able to scp to that host without a password
  --overwrite            File will be overwritten each time (not default)

  --createfile #         Create a # GB file to clear the memory buffer


theoretical module     // Produce a theoretical curve for IB or Ethernet
==================

    These formulas have the packet header sizes as accurate as I've been
able to find.  You can add in a rendezvous threshold to try to duplicate
MPI performance though it won't get the small message shape correct since
MPI libraries perform several tricks to get faster performance.

make theo
NPtheo --ethernet   --maxrate 40 --latency 15
NPtheo --roce       --maxrate 40 --latency 1.3
NPtheo --infiniband --maxrate 32 --latency 1.3

    (must choose ethernet|infiniband|roce with maxrate and latency)
  --ethernet     Use 1500 byte mtu size, 26 byte preamble, 20 byte header
  --infiniband   Use 4096 byte mtu size, 30 byte header
    --alliance     Start with 2 padded half packets
    --mellanox     Start with a padded half packet
  --roce         Use 4096 byte mtu size, 74 byte header
  --maxrate #    The theoretical max bandwidth for the medium in Gbps
  --latency #    The measured small message latency in microseconds
  --mtu #        Change the MTU size (dafault 1500 Ethernet 4096 IB)
  --inline #     The first # bytes are inlined with the message
  --rendezvous # Add an extra message send at the rendezvous threshold


shmem module     // I have not tested the shmem module in a while
============


