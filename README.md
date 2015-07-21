# CMU-assembly-challenge

## What is this?

This is my walkthrough for the CMU binary bomb challenge. The exercise is as
follows: you are given a binary, whose source you do not have, and when you
run this binary it expects to receive certain inputs, otherwise it 'blows' up.
I disassembled the binary and reserve engineered it to figure out the right
inputs. The original binary given to CMU students lowers your grade every time the
bomb is set off but I managed to get my hands on a modified version that, as far
as I can tell, does nothing when set off.

## Why?

I did this because I wanted to learn basic assembly. I found the 'Intro to x86
assembly' course on opensecuritytraining.info and this exercise was included
as a final project. Also, because reserve engineering a binary is fun!

## Tools

I used gdb and objdump. The binary is included in this repo and so is its
disassembly.

## Notes

The binary was compiled on a 32-bit machine. I am running on 64-bit and so
my gdb functions for displaying the stack had to be adjusted. If you want to
display your stack make sure you have a 4 byte offset between each word.
Something like the following should do the trick:

```
define display-stack
  display/dw $sp
  display/dw $sp + 4
  display/dw $sp + 8
  display/dw $sp + 12
  display/dw $sp + 16
  display/dw $sp + 20
end
```

## Level 1

So we want to start from the `<main>` tag, whose code gets called after
various initialization gets done. From a quick glance at `<main>` we can
see that it calls a number of `<phase_n>` methods along with
`<initialize_bomb>`. It also calls out to `<fopen>`, which presumably opens
stdin. If we look at `initialize_bomb` we can see a call to `<signal>`, which
as you might guess sets up signal handling for the binary. Naturally, after
reading this I tried to Ctrl-C the binary, which caused the binary to print
a cute taunting message and a follow up Ctrl-C actually disarmed it.

`<phase_1>` is pretty straight forward. It calls to `<strings_not_equal>`,
which takes two pointers to strings on the stack, one of those is the
string you provided (read in with a call to `<readline>` before `<phase_1>`)
and the other has a hard-coded address of `0x80497c0`. If you examine
`x/s 0x80497c0` you get 'Public speaking is very easy.' and bam that's your
answer.

## Level 2

`<phase_2>` starts with a call to `<read_six_numbers>`, which
is just a wrapper around `<sscanf>`. The format string is hardcoded, is
located at `0x8049b1b` and expects six integers separated by whitespace. Space
for those integers was allocated on the stack prior to the call to
`<read_six_numbers>` by subtracting from the ESP.

After entering your six integers you are faced with a loop. The loop index
is EBX and EAX contains a kind of running sum whose value is compared to your 
integers at each iteration. The loop looks something like the following:

```
ebx = 1
eax = ebx + 1
eax *= (ebx - 1)th element
eax == ebx th element
```

If you trace out this loop you get the following six integers: 1 2 6 24 120 720.

## Level 3

`<phase_3>` starts out with an `<sscanf>` as well. The format string,
hardcoded at `0x80497de`, expects an integer, character, integer separated by
single whitespace. After your input is correctly read, the binary ensures
that the first argument is less than 7.

The first argument you provide is used as an offset into an array of
addresses (see instruction at `0x8048bd6`). The code jumps to the address
contained at that offset, essentially providing the functionality of a switch
statement. I decided to try out with 1 as the first argument and that got me
to `0x8048c00`. At that point, in order to proceed EBP-4 must be equal to
0xd6 or 214 in decimal. EBP-4 is the third argument provided.

After getting through this, the code jumps to `0x8048c8f` (note that all
other branches of the switch also redirect there),
which is where the second argument you provided is compared to the lower
16-bits of EBP aka BL. In our branch, BL got set to 0x62 or 98 in decimal or
the character 'b'. Thus, the correct input is '1 b 214'.

## Level 4

`<phase_4>` starts off with an `<sscanf>` with a format string of one integer.
That integer gets passed on to `<func4>`, whose return value is, by convention,
stored in EAX and EAX needs to equal `0x37` or 55 in decimal.

`<func4>` is a recursive function. After staring at the assembly for a while
I figured out the base case and the recursive call. The code looks something
like the following:

```python
def func4(x):
    if x <= 1:
        return 1
    else:
        return func4(x - 1) + func4(x - 2)
```

This looks an awful lot like the Fibonacci sequence. I ran the above code
through a Python interpreter and quickly figured out that an input of
9 gives a result of 55 and in fact 55 is the 9th element in the Fibonacci
sequence if you start from 1.

## Level 5

The input to `<phase_5>` must be a six character string. The code then
iterates over the input string, does some processing on each character,
and then compares the result to the expected result, namely the string
'giants'. There's also a magical constant string `isrveawhobpnutfg` located
at `0x804b220` whose purpose will be explained shortly.

At each iteration, the character from our string gets AND-ed
with `0xF`, which has the effect of preserving the character's least significant
4 bits and setting to zero the character's most significant 4 bits. For example
if our character is 'O' (`0x4F`) then AND-ing it with `0x0F` will produce `0x0F`.

The result of the AND is then used as an index into the mysterious string above
so that we spell out `giants`. Here's a mapping of each character in `giants` to
that character's position in the mysterious string:

```
15 0 5 11 13 1
g  i a n  t  s
```

So our input string needs to have characters all starting from the same base
such that each character's offset from that base is equal to the offsets above.
I arbitrarily picked `0x40` ('@') as that base. Therefore, the first character
of the input string will be `0x40` + 15 (decimal) = `0x4F` ('O'). Continuing this
process we come to the correct answer of 'O@EKMA'.

## Level 6

This level was really hard! I won't go into the various loop details because that
would take too much time. `<phase_6>` expects 6 integers, separated by whitespace,
all less than 6. After staring at the assembly for a while I realized that
the code is re-arranging pointers to structs. We start out with a linked list and in
order to complete the level we need to arrange that list in a particular way by
specifying that order using our input integers.
The structs in the list look something like the following:

```
struct node {
    int val;
    int num;
    struct node* next;
}
```

The `val` integer field is the field by which we need to sort the nodes. I traversed
the list and got the following values for each node:

|node #| unsigned val| signed val|
|:----:|:-----------:|:---------:|
|node 1|253|-3|
|node 2|725|725|
|node 3|301|301|
|node 4|997|997|
|node 5|212|-4|
|node 6|432|432|

The input integers are the order in which the nodes need to be arranged. So if the
first integer is 4 then that means that the fourth node will be first, and so on.
We need to arrange the nodes in decreasing order based on a signed comparison
of the `val` field. Thus looking at the above table the correct ordering is
'4 2 6 3 1 5'.

## Hidden phase

I unfortunately didn't get a chance to get to this phase but maybe after a short
break from assembly I'll come back and try to solve it.


Thanks for reading! Feel free to message me if you have any questions.