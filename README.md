# Reproducing bug when using HashAlgorithm.crc16_custom consecutive times

## Introduction

The objective of this tutorial is to extend a basic calculator
from P4 tutorials to reproduce a bug when using HashAlgorithm.crc16_custom in
consecutive hash operations.

The header will contain an operation to perform and two operands.
When a switch receives a calculator packet header, it will execute the
operation on the operands, and return the result to the sender.

We will use the following header format:

             0                1                  2              3
      +----------------+----------------+----------------+---------------+
      |      P         |       4        |     Version    |     Op        |
      +----------------+----------------+----------------+---------------+
      |                              Operand A                           |
      +----------------+----------------+----------------+---------------+
      |                              Operand B                           |
      +----------------+----------------+----------------+---------------+
      |                              Result                              |
      +----------------+----------------+----------------+---------------+


-  P is an ASCII Letter 'P' (0x50)
-  4 is an ASCII Letter '4' (0x34)
-  Version is currently 0.1 (0x01)
-  Op is an operation to Perform:
 -   '%' Result is a hash using OperandA as input as described below in
 operation_crc() function. OperandB is not used. It performs a single
 hash operation.
 -   '#' Result is a hash using OperandA as input as described below in
 operation_dcrc function. It performs two consecutive hash operations. OperandB
 is used as input for the first hash operation, and OperandA is used as input
 for the second hash operation. The two hash operations should be independent.

```
action operation_crc() {

    bit<32> nbase=0;
    bit<64> ncount=4294967296*2;
    bit<32> nselect;
    bit<32> ninput = hdr.p4calc.operand_a;

    hash(nselect,
    HashAlgorithm.crc16_custom,
    nbase,
    {ninput},ncount);

    send_back(nselect);
}
```

```
action operation_dcrc() {

    bit<32> nbase=0;
    bit<64> ncount=4294967296*2;
    bit<32> nselect;
    bit<32> ninput = hdr.p4calc.operand_b;

    hash(nselect,
    HashAlgorithm.crc16_custom,
    nbase,
    {ninput},ncount);

    bit<32> nbase2=0;
    bit<64> ncount2=4294967296*2;
    bit<32> nselect2;
    bit<32> ninput2 = hdr.p4calc.operand_a;

    hash(nselect2,
    HashAlgorithm.crc16_custom,
    nbase2,
    {ninput2},ncount2);

    send_back(nselect2);
}
```


## Step 1: Run the (incomplete) starter code

As a first step, compile the incomplete `calc.p4` and bring up a
switch in Mininet to test its behavior.

1. In your shell, run:
   ```bash
   make
   ```
   This will:
   * compile `calc.p4`, and

   * start a Mininet instance with one switches (`s1`) connected to
     two hosts (`h1`, `h2`).
   * The hosts are assigned IPs of `10.0.1.1` and `10.0.1.2`.

2. You access s1 directly from the Mininet command prompt:

```
mininet> xterm s1
>
```
3. From s1, start the Command Line Interface (CLI) for the software switch:

```
# simple_switch_CLI
```
You should get a RunTimeCmd prompt:
```
Obtaining JSON from switch...
Done
Control utility for runtime P4 table manipulation
RuntimeCmd:
```
Configure the CRC16 parameters:
```
RuntimeCmd: set_crc16_parameters calc 0x002b 0x0 0x0 false false
```
4. We've written a small Python-based driver program that will allow
you to test your calculator. You can run the driver program directly
from the Mininet command prompt:

```
mininet> h1 python calc.py
>
```

5. The driver program will provide a new prompt, at which you can type
basic expressions. The test harness will parse your expression, and
prepare a packet with the corresponding operator and operands. It will
then send a packet to the switch for evaluation. When the switch
returns the result of the computation, the test program will print the
result.
```
> 1#6
1#6
49351
> 1%6
1%6
49
>
```

 Both results should return the same value, but it is not working correctly.

A interesting observation: if you perform the same operation without
setting up a custom crc configuration beforehand the bug does not appear:
```
> 1#6
1#6
49351
> 1%6
1%6
49351
```
