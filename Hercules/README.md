## Hercules

A mathematical one.

![](1.png)


## Solution

After connecting, the program exits almost immediately, only printing two lines…

![](2.png)

...that are somewhat randomized each time. 

![](3.png)

The first line is probably the encrypted flag, and retrieving the original seems to be the objective. The trio of large numbers is daunting, and the challenge description is vague, so let’s just open up the binary in Ghidra.

![](4.png)

The file is stripped, but we can find our `main` by looking inside the `entry` function:

![](5.png)

(Renaming functions and variables as we go, for clarity.)

![](6.png)

The hardcoded bytes immediately stand out as a char array. There are better ways to decipher this, but if Ghidra isn't feeling up to it, throwing them into any hex-to-ASCII works fine:

![](7.png)

This proves to be too easy, of course:

![](8.png)

Worth the shot.

![](9.png)

The real flag is in the executable running on the server, and we need to learn how to decrypt its output.

We can see that main calls a new function with `0x19` and the flag as its parameters:

![](10.png)

That's a lot of variables. However, we may notice that the variables in lines 27-29 actually form an array, which the decompiler hasn't recovered properly. Reading line 65 gives a clue to that, as
`local_88`, which holds the address of the first variable, is accessed as `local_88[1]` and `local_88[2]`, and reading the disassembly confirms these suspicions.

Similarly, `local_58` seems to be the start of an array of 9 variables.

We can make the important part of the function more legible:

![](11.png)

All that happens is that one of `f0`, `f1` and `f2` is randomly selected and called with the larger array as parameter, followed by two more functions. (Ignore the fact
that those seemingly have three arguments here, that is a decompiler artifact. It is later clear that they only take two.) From `main`, we know that this is repeated `25` times. After that, the encrypted flag is printed, followed by the smaller array.

Comparing this to the output of the binary, it seems that `fun3` must be the one encrypting the flag, while `fun2` is probably making the inital {3,4,5} array larger.

Let's analyze `f0, f1, f2` first:

![](12.png)

It looks rather disgusting at first, but after retyping the function argument from `long` to the `long*` that we know it accepts, and beautifying a bit:

![](13.png)

A better look at the arithmetic involves suggests that our array is better interpreted as a 3x3 matrix. We see that `f0` makes sure the 2nd column of the matrix is negative, and the other two positive.

The other two functions perform similar tasks (with `arr9` renamed to `matrix` for clarity):

![](14.png)

![](15.png)

`f1` is turning every column positive, while `f2` is setting only the first column to negative. 
Therefore, what does the matrix look like after `f0`, `f1` and `f2` respectively:

![](16.png)

Continuing to `fun2`:

![](17.png)

The important part, after retyping the parameters:

![](18.png)

For each `i`-th column in the matrix, calculate some `operation` and write it in the `i`-th entry of a new array? This looks suspiciously like matrix multiplication.

![](19.png)

Yep, `operation` is nothing more than scalar product. So, this function takes our triplet of numbers, and multiplies it with a random one of our matrices.

Let's look at the final function:

![](20.png)

It seems to be a sort of shift cipher, shifting the value of every character in the flag by the value of the corresponding number in the array.

Now, to recap how the entire program works:
- We begin with our flag and the array {3,4,5}.
- A loop is repeated 25 times, where:
  - The array is multiplied with a randomly selected matrix out of the three known.
  - The array is used as a key to encrypt the flag.
- The encrypted flag and final array are then given to us. 

We don't mess with /dev/urandom, but we don't have to - there are "only" 3^25 different ways that these matrices may be selected, and that is bruteforcable.

However, in the spirit of `From his foot, we can measure him`, some might find the following solution a bit more beautiful:

Let us assume that the challenge author is not evil, and there really is a way to find the entire sequence of matrices involved, from the final one given.

By definition, the way to go about reversing matrix multiplication is finding the inverses:

![](21.png)

They do look similar. But which one to use? Bruteforcing all the inverses to find what sequence brings us back to {3,4,5} could take as much as 3^25 tries, unless... 

![](22.png)

...they do not need to be bruteforced.

![](14.png)



#### flag = `dctf{f1rst_step_t0wards_b3ll_l4bs}`