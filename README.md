# CMU-assembly-challenge

## What is this?

This is my walkthrough for the CMU binary bomb challenge. The exercise is as
follows: you are given a binary, whose source you do not have, and when you
run this binary it expects to receive certain inputs, otherwise it "blows" up.
I disassembled the binary and reserve engineered it to figure out the right
inputs. The original binary given to CMU students lowers your grade every time the
bomb is set off but I managed to get my hands on a modified version that, as far
as I can tell, does nothing when set off.

## Why?

I did this because I wanted to learn basic assembly. I found the "Intro to x86
assembly" course on opensecuritytraining.info and this exercise was included
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
`x/s 0x80497c0` you get "Public speaking is very easy." and bam that's your
answer.

## Level 2

`<phase_2>` starts with a call to `<read_six_numbers>`, which it turns out
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
Pretty easy, right?

## Level 3

`<phase_3>` starts out with a `<sscanf>` as well. The format string,
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
the character 'b'. Thus, the correct input is "1 b 214".