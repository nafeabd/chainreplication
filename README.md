
================
Chain Replication Using DistAlgo

OVERVIEW

Chain replication is implementation as described in

  [vRS2004chain] Robbert van Renesse and Fred B. Schneider.  Chain
  Replication for Supporting High Throughput and Availability.  In
  Proceedings of the 6th Symposium on Operating Systems Design and
  Implementation (OSDI), pages 91-104.  USENIX Association, 2004.

============================================================
FUNCTIONALITY OF THE REPLICATED SERVICE

The replicated service stores bank account information.  it stores the
following information for each account: (1) balance, (2) sequence of
processed updates, denoted processedTrans.

the service supports the following query and update requests.  the reply to
each request is an instance of the class Reply, shown below as a tuple for
brevity.

enum Outcome { Processed, InconsistentWithHistory, InsufficientFunds }

class Reply {
  string reqID;
  string accountNum;
  Outcome outcome;
  float balance;
}

note: making request identifiers strings (rather than numbers) allows them
to have structure, e.g., "1.1.1".  this makes it easier to generate unique
request identifiers, using a structure such as
bankName.clientNumber.sequenceNumber.

QUERIES

  getBalance(reqID, accountNum): returns <reqID, Processed, balance>.

UPDATES

  deposit(reqID, accountNum, amount): if a transaction with this reqID has
  not been processed for this account, increase the account balance by the
  amount, append the transaction to processedTrans, and return <reqID,
  accountNum, Processed, balance>, where balance is the current balance.
  if exactly the same transaction has already been processed, re-send the
  same reply.  if a different transaction with the same reqID has been
  processed for this account, return <reqID, accountNum,
  InconsistentWithHistory, balance>.

  withdraw(reqId, accountNum, amount): if a transaction with this reqID has
  not been processed for this account, and the account balance is at least
  the amount, decrease the account balance by the amount, append the
  transaction to processedTrans, and return <reqID, accountNum, Processed,
  balance>.  if a transaction with this reqID has not been processed for
  this account, and the account balance is less than the amount, return
  <reqID, accountNum, InsufficientFunds, balance>.  if exactly the same
  transaction has already been processed, re-send the same reply.  if a
  different transaction with the same reqID has been processed for this
  account, return <reqID, accountNum, InconsistentWithHistory, balance>.


  transfer(reqID, accountNum, amount, destBank, destAccount): if a
  transaction with this reqID has not been processed for this account, and
  the account balance is at least the transfer amount, then transfer the
  requested amount of funds from the specified account in this bank to
  destAccount in destBank, and return <reqID, accountNum, Processed,
  balance>.  if a transfer with this reqID has not been processed for this
  account, and the account balance is less than the transfer amount, return
  <reqID, accountNum, InsufficientFunds, balance>.  if exactly the same
  transaction has already been processed, re-send the same reply.  if a
  different transaction with the same reqID has been processed for this
  account, return <reqID, accountNum, InconsistentWithHistory, balance>.


============================================================
SIMPLIFICATIONS

if an account mentioned in a request does not already exist, then it is
automatically created, with initial balance 0, and then the request is
processed as described above.

assume clients never send requests with non-positive amounts.

assume the master never fails.  therefore, you do not need to implement
Paxos.

the same master is used by all banks.  the master reports every failure to
all servers of all banks.

assume that the pattern of failures is limited so that there is always at
least one living server for each bank.

it is sufficient to store all information in RAM.  it's OK that information
stored by a process is lost when the process fails or terminates.

it is sufficient to test your system with all of the clients being threads
in a single process.  you should run testcases with up to 3 banks and up to
6 clients per bank.

============================================================
NETWORK PROTOCOLS

as stated in the paper, communication between servers should be reliable,
so use TCP as the underlying network protocol for it, and assume that TCP
is reliable.  this applies to communication between servers in the same or
different chains.  also, communication between the master and servers is
reliable, so use TCP for it.

communication between clients and servers may be unreliable, so use UDP as
the underlying network protocol for it.  for simplicity, assume that each
request or reply fits in a single UDP packet.

============================================================ 
CONFIGURATION FILE

all clients, servers, and the master read information from a configuration
file whose name is specified on the command line.  for simplicity, all
three kinds of processes read the same configuration file, and each kind of
process ignores the information it does not need.  note: this implies that
configuration file contains information for all banks.

the configuration file contains enough information to specify a testcase;
thus, you can run different testcases simply by supplying different
configuration files.  information in the configuration file includes (but
is not limited to): the names of the banks, the length of the chain for
each bank, Internet addresses and port numbers of the master and servers
(for all banks), the number of clients of each bank, a description of the
requests submitted by each client (explained below), server
startup delays, and server lifetimes (explained below).

The DistAlgo implementation should ignore the IP
addresses and port numbers in the configuration file.

the configuration file should have a self-describing syntax.  instead of
each line containing only a value, whose meaning depends on the line number
(where it is hard for readers to remember which line number contains which
value), each line should contain a label and a value, so the reader easily
sees the meaning of each value.

the description of a client's requests in the configuration file can have
one of two forms: (1) a single item random(seed, numReq, probGetBalance,
probDeposit, probWithdraw, probTransfer), where seed is a seed for a
pseudo-random number generator, numReq is the number of requests that will
be issued by this client, and the remaining parameters (which should sum to
1) are the probabilities of generating the various types of requests.  (2)
a sequence of items, each representing one request using some readable and
easy-to-parse syntax, such as "(getBalance, 1.1.1, 46)".  the configuration
file should also contain separate entries controlling the following: the
duration a client waits for a reply, before either resending the request or
giving up on it; the number of times a client re-sends a request before
giving up on it; whether the client re-sends a request to the new head for
a bank, if the client, while waiting for a reply from that bank, is
notified by the master of failure of the head for the bank.

the configuration file specifies a startup delay (in milliseconds) for
each server.  a server's main thread does essentially nothing except sleep
until the server's startup delay has elapsed.  this feature facilitates
testing of the algorithm for incorporating new servers (i.e., extending the
chain).  servers that are part of the initial chain have a startup delay of
zero.

the configuration file specifies a lifetime for each server, which may be
(1) "receive" and a number n (the server terminates immediately after
receiving its n'th message), (2) "send" and a number n (the server
terminates immediately after sending its n'th message), or (3) "unbounded"
(the server never terminates itself).  furthermore, the number n in (1) and
(2) can be replaced with the string "random", in which case the server
generates a random number in a reasonable range, outputs it to a log file,
and uses that number.


============================================================
LOGS

every process should generate a comprehensive log file describing its
initial settings, the content of every message it received, the content of
every message it sent, and every significant internal action it took.
every log entry should contain a real-time timestamp.  every log entry for
a sent message should contain a send sequence number n, indicating that it
is for the n'th message sent by this process.  every log entry for a received
message should contain a receive sequence number n, indicating that it is
for the n'th message received by this process.  (send and receive sequence
numbers are useful for choosing server lifetimes that correspond to
interesting failure scenarios.)  the log file should have a self-describing
syntax, in the same style as described for the configuration file.  for
example, each component of a message should be labeled to indicate its
meaning.

============================================================
FAILURE DETECTION

chain replication is designed to work correctly under the fail-stop model,
which assumes perfect failure detection (cf. [vRG2010replication, section
2.4]).  there are two completely different approaches to handling this.

one approach, suitable during development and testing, is to "fake" failure
detection.  specifically, when a server is preparing to terminate as
specified by its configuration, it sends a message announcing its
termination to the master.  with this approach, the master implements
failure detection simply by waiting for such notifications.  this
approach's advantage is guaranteed absence of false alarms.  this
approach's disadvantage is that unplanned failures (e.g., a process
terminating due to an uncaught exception) are not detected.

the other approach is to implement real failure detection, using timeouts.
for example, each server S sends a UDP packet containing S's name to the
master every 1 second.  every (say) 5 seconds, the master checks whether it
has received at least 1 message from each server that it believes is alive
during the past 5 seconds.  if not, it concludes that the uncommunicative
servers failed.  note that a few occasional dropped UDP packets should not
cause the master to conclude that a server has failed.  this approach never
misses real failures, but it can produce false alarms if enough packets are
dropped or delayed.

Two ways to get a server process to periodically send heartbeat
messages to the master in DistAlgo.

1. create a separate thread to do this.  this is not especially difficult
in DistAlgo.  DistAlgo does not require any special treatment of threads.
use standard Python thread commands.

2. send the heartbeat messages in the main thread.  for example, the method
could look like this:

while True:
  if (await(False)): pass
  elif timeout(T): send heartbeat message

the process will block at the await statement, running receive handlers
when messages arrive, until time T has elapsed, at which time the process
will send a heartbeat message, and then loop back to the await statement.

similar code can be used in the master to periodically "wake up" and check
whether it has recently received enough heartbeat messages from each
server.

This Project is done as part of CSE535-Asynchronous Systems course work - Stony Brook University for Fall 2014 semester . 
Most of the above description is copied from official description provided for the project for above mentioned course.


