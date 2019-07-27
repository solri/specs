# Computation Strategies

In this section, we list some common computation strategies for
on-chain processing.

## Stop-the-world Computation

Stop-the-world computation represents a kind of computation where we
need to iterate over the full account list. In this case, because the
account list can be huge, the computation itself is
expensive. However, we must ensure block processing takes minimal time
to ensure network safety and to ensure proof of work cannot be
exploited. There are two methods to do this:

* **State copying and churn limit**: whenever we need to do
  stop-the-world computation, copy the current state into a new
  structure, and process accounts with a churn limit. In this way, we
  process the whole computation over the span of potentially hundreds
  of blocks, which can last a day or two.
* **Data migration**: when an account is touched, process the
  computation and migrate the account into a new data structure. In
  this way, we always know what accounts are not processed and what
  accounts are processed, by checking whether it is in the old or new
  data structure.
