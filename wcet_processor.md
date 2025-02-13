# Predictable Processor 

## Goal

Statically ensure the predictablity of the WCET of a processor but also want to enjoy the benefits of optimizations for throughput when deadline can be met.

## [VISA: Virtual Simple Architecture](https://ericrotenberg.wordpress.ncsu.edu/architectures-for-real-time)

Early 2000s.


Dual mode pipeline design: a complex one(for throughput) and a simple one(for analysis). [Default complex mode, switch to simple mode if a checkpoint is missed.](https://ericrotenberg.wordpress.ncsu.edu/files/2022/08/conference_ISCA-30.pdf) 

## [Pret Architecture](https://www.cs.columbia.edu/~sedwards/papers/lickly2008predictable-tr.pdf)

More sophisticated. Not just deadline guarantees, but also detailed timing.

## [FlexPRET: A Processor Platform for Mixed-Criticality Systems](https://www2.eecs.berkeley.edu/Pubs/TechRpts/2013/EECS-2013-172.pdf)

"The FlexPRET CPU architecture is the third generation of multi-threaded PRET machines designed at UC Berkeley".

## [InterPRET: a Time-predictable Multicore Processor](https://dl.acm.org/doi/pdf/10.1145/3576914.3587497)

FlexPRET + Network On Chip





