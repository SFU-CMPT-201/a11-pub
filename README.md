# Assignment 11: A Simple Group Chat Server with Fuzzing Clients

In this assignment, you will write a simple group chat server program and a client program. The
server will accept multiple clients and relay client messages back to the clients. Each client, on
the other hand, will be a fuzzer generating messages and sending and receiving those messages to and
from the server.

## Group Chat Server

The group chat server accepts client connections, receives client messages, and echoes those
messages back to all clients. The following are the requirements of the server.

* The command-line interface should be able to start the server as follows.

  ```bash
  $ ./server <port number> <# of clients>
  ```

* The executable of your server should be named `server`.
* The server should bind to port `port number`.
* `<# of clients>` is the expected number of clients.
    * A client can join any time and leave any time.
    * You do not need to handle cases where this number is just too big and our VM cannot handle
      that many clients. We will not test with more than 100 clients.
* The server should use `AF_INET`.
* The server should bind to all available IP addresses for the local host.
* The server should receive all messages from each and every client as they send them.
* When the server receives a message from a client, it should send the message to *all* clients,
  *including* the sender.
* Message ordering
    * When a client sends multiple messages, all clients should receive the messages in the original
      sending order.
    * In addition, all clients should receive all messages in the same order.
        * For example, suppose we have three clients, C0, C1, and C2.
        * Consider the following scenario.
            * C0 sends a message, M0, which is received by the server.
            * C1 then sends a message, M1, which is also received by the server.
            * Lastly, C2 sends a message, M2, which is again received by the server.
        * Based on the above scenario, C0, C1, and C2 should all receive M0 first, M1 next, and then
          M2.
    * This message ordering is not a trick requirement. It is just what you expect from a group chat
      room.
* Note: the simplest way to test a server is to use `telnet <IP> <port>`.
    * Suppose your local IP address is `192.168.68.6` and you run the server, which binds to port
      `8000`.
    * `telnet 192.168.68.6 8000` will act as a client connect to the server.
    * `127.0.0.1` is a local IP address that refers to the local machine, so `telnet 127.0.0.1 8000`
      will also work.
    * `localhost` is a host name that refers to the local machine, so `telnet localhost 8000` works
      as well.
    * You can enter messages on your terminal as the standard input to `telnet` and they will be
      sent to the server.

## Messaging Protocol

The following is how the server and the clients should format and interpret messages they send and
receive.

* For clarity, we use two terms---a *group chat message* and a *message*.
    * A *group chat message* refers to what we would normally call a "message" in the context of a
      group chat app.
    * A *message* is a piece of data sent by a client or a server through a socket. As described
      below, this may or may not contain a group chat message.
* The message size should be no more than 1024 bytes.
* Both for the server side and the client side, a message should always start with a one-byte
  (`uint8_t`) "type."
    * `0` means that it is a regular message that contains a group chat message.
        * We refer to this as a type `0` message.
    * `1` means that it is a special "end of execution" message.
        * [Two-Phase Commit Protocol](#two-phase-commit-protocol) below fully explains this.
        * We refer to this as a type `1` message.
* Both for the server and the clients, a new line character `'\n'` always marks the end of a
  message.
    * However, you need to be careful with this due to the requirement described next.
* When the server receives a type `0` message from a client, the server should get the *sender's*
  address (IP and port) and include the address in the message sent to all clients.
    * Consider a typical group chat app. When a client sends a group chat message, everyone should
      know who sent it. We achieve this by having the server add the sender's address to each group
      chat message.
    * The IP address should be of type `uint32_t`. It should come after the first byte that
      indicates the type (`0`).
    * The port number should be of type `uint16_t`. It should come after the IP address.
    * Note that only the server performs this address inclusion. A client does not do this.
    * Also note that the address and the port number are numbers, `uint32_t` and `uint16_t`,
      respectively. They are not strings. This means that those numbers can contain a byte value
      that is `'\n'`. You need to be careful with this when you send and receive messages.
* In summary, when the server receives a message from a client, it should interpret the first 1 byte
  as the type. If it is `0`, then the server should interpret the rest of the message as a group
  chat message until it reads `'\n'`.
    * If it is `1`, the server should follow the description in [Two-Phase Commit
      Protocol](#two-phase-commit-protocol) below.
* Similarly, when a client receives a message from the server, it should interpret the first 1 byte
  as the type. If it is `0`, then the client should interpret the next 4 bytes as the original
  sender's IP address, the next 2 bytes as the original sender's port number, and the rest as a
  group chat message until it reads `'\n'`.
    * If it is `1`, the client should follow the description in [Two-Phase Commit
      Protocol](#two-phase-commit-protocol) below.

## Fuzzing Clients

The client program is a simple custom fuzzer that generates messages and sends those messages to the
server. It also receives messages sent by the server. It is a custom fuzzer in the sense that it
does not use a fuzzing framework such as [libFuzzer](https://llvm.org/docs/LibFuzzer.html) discussed
in A5. The following are the requirements.

* The command-line interface should be able to start a client as follows.

  ```bash
  $ ./client <IP address> <port number> <# of messages> <log file path>
  ```

* The executable should be named `client`.
* `IP` and `port` are the IP address and the port number of the server, respectively.
* A client should follow the above [message protocol](#messaging-protocol) when sending and
  receiving messages.
* After it starts, a client should keep generating and sending random messages. `<# of messages>`
  should be the number of messages sent.
    * The random messages should be type `0` messages.
    * After sending `<# of messages>` messages, the client should stop generating and sending type
      `0` messages, and prepare for termination as described below in [Two-Phase Commit
      Protocol](#two-phase-commit-protocol).
    * You can use `getentropy()` to generate random bytes. Read `man getentropy` to understand how
      to use it.
    * Since `getentropy()` gives you random bytes that you may not be able to print out, use the
      following function to convert it to a hex string.

      ```c
      #include <stdint.h>
      #include <stdio.h>
      #include <unistd.h>

      /*
       * `buf` should point to an array that contains random bytes to convert to a
       * hex string.
       *
       * `str` should point to a buffer used to return the hex string of the random
       * bytes. The size of the buffer should be twice the size of the random bytes
       * (since a byte is two characters in hex) plus one for NULL.
       *
       * `size` is the size of the `str` buffer.
       *
       * For example,
       *
       *   uint8_t buf[10];
       *   char str[10 * 2 + 1];
       *   getentropy(buf, 10);
       *   convert(buf, str, 21);
       */
      void convert(uint8_t *buf, char *str, ssize_t size) {
        if (size % 2 == 0)
          size = size / 2 - 1;
        else
          size = size / 2;

        for (int i = 0; i < size; i++)
          sprintf(str + i * 2, "%02X", buf[i]);
      }
      ```

* A client should print out each and every type `0` message it receives from the server and store
  them in a file named `<log file path>`.
    * Use this format: `"%-15s%-10u%s"`, e.g., `printf("%-15s%-10u%s", ip, port, str);`
    * For example, `192.267.128.205     9000      9391DE3E275ADB19637D`
    * Make sure you print out one message per line. Adjust the format string accordingly regarding
      the new line character.
* Note that a client should be able to send messages and receive messages concurrently.

## Two-Phase Commit Protocol

To gracefully end the execution for both the server and the clients, the server and the clients
should collectively implement a simplified [two-phase commit
protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) signaling the end of execution as
follows.

* After sending `# of messages` messages, each client should send a type `1` message to the server
  (as opposed to a type `0` message described in [Message Protocol](#messaging-protocol)). This is
  to indicate that the client is done sending messages.
* After the server receives a type `1` message from each and every client, it should send a type `1`
  message to all clients. It should then terminate itself.
* Each client should terminate after receiving a type `1` message from the server.

## Code Structure and CMake

* You need to use the same code structure as previous assignments with `src/` and `include/`.
* You also need to write `CMakeLists.txt` that produces two executables, `server` and `client`.
* You need to set the `CC` and `CXX` environment variables to `clang` and `clang++`. If you haven't,
  open your ~/.zshrc and add the following two commands at the end.

  ```bash
  export CC=$(which clang)
  export CXX=$(which clang++)
  ```

## Grading

* There are two executables `server-tester` and `client-tester` under `bin/` that you can use to
  test your program. These executables are essentially the same as the ones we use for grading,
  though they may not be exactly the same.
* You *should not* use these executables as debuggers. They do not tell you what the problems are.
  They do provide some error messages but those are not intended to serve as debug messages. They
  mostly just provide only a simple pass/fail result.
* When a test case does not pass, you need to construct the same scenario and debug your program to
  find the problems.
* Testing your server program: `server-tester` is the program that tests your server program. The
  following is how you run the program.

  ```bash
  $ ./server-tester <IP address> <port number> <# of clients> <# of messages> <test #>
  ```

    * You need to run your server program first before running `server-tester`.
    * `IP address` and `port number` are the IP address and the port number of the server,
      respectively.
    * `<# of messages>` is the number of messages sent by each client. Some tests may ignore this
      argument as they do not need more than one message.
    * `<# of clients>` is the number of clients that start. Some tests may ignore this argument as
      they do not need more than one client.
    * `<test #>` is the test number. The following are the test numbers and their descriptions.
        * `0` [5 pts]: One client sends one type `0` message and receives some message back.
        * `1` [10 pts]: One client sends one type `0` message and receives the server's response
          message back in the correct format (i.e., according to the protocol with 0, IP, port, and
          chat message).
        * `2` [10 pts]: One client keeps sending type `0` messages and receives those messages back
          in the correct format and in the original sending order. We will test this with a large `#
          of messages` (e.g., 1000).
        * `3` [5 pts]: One client sends one type `1` message and receives one type `1` message back
          from the server. The server terminates.
        * `4` [10 pts]: Many clients each send one type `0` message and receive all the messages
          back in the correct format and in the same order. We will test this with a large `# of
          clients` (e.g., 100).
        * `5` [15 pts]: Many clients each keep sending messages and receiving all the messages back
          in the correct format and in the same order. We will test this with a large `# of clients`
          and a large `# of messages` (e.g., 100 clients and 100 messages each).
        * `6` [5 pts]: Many clients each send one type `1` message and all receive one type `1`
          message back from the server. The server terminates.
* Testing your client program: `client-tester` is the program that tests your client program. The
  following is how you run the program.

  ```bash
  $ ./client-tester <client executable path> <client log prefix> <port number> <# of clients> <# of messages> <test #>
  ```

    * This is essentially a server but it also executes your client. Thus, the output of your client
      program will be printed out on the terminal, along with the output of the program.
    * `<client file path>` is the path to your client executable.
    * `<client log prefix>` is the prefix of the log file that your client program generates. The
      program will append the client number to the prefix to create the log file name.
    * `port number` is the port number that `client-tester` binds to.
    * `<# of clients>` is the number of clients that you want to start. Some tests may ignore this
      argument as they do not need more than one client.
    * `<# of messages>` is the number of messages sent by each client. Some tests may ignore this as
      they do not need more than one message.
    * `<test #>` is the test number. The following are the test numbers and their descriptions.
        * `0` [5 pts]: After a single client starts, the server receives at least one type `0`
          message and the client prints the message out correctly.
        * `1` [10 pts]: After a single client starts, the server receives `<# of messages>` random
          type `0` messages and the client prints out all messages correctly.
        * `2` [5 pts]: After a single client sends `<# of messages>` messages, the server receives
          one type `1` message and sends it back. The client terminates.
        * `3` [15 pts]: After many clients start, the server keeps receiving type `0` messages and
          the clients print out all messages correctly.
        * `4` [5 pts]: After receiving `<# of messages>` from each and every client, the server
          receives one type `1` message from them and sends it back. All clients terminate.
* Code that does not compile with CMake gets a 0.
* Code that does not generate all the required executables (`client` and `server`) gets a 0.
* Memory issues have a penalty of -10 pts. Use appropriate sanitizers and enable the debug option.
  Make sure your code works without the debug option as well. Before you submit, make sure that you
  remove the debug option.
* A wrong code directory structure has a penalty of -10 pts.
* Thread/synchronization issues have a penalty of -10 pts.
* You should not hard-code or customize your implementation tailored to our test cases. Generally,
  you should consider the provided test cases as examples or specific instances of general cases.
