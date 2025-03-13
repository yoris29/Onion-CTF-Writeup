# 0nion (!mario)
- Once we download the assets we get a .yaml file containing informations about the ctf and telling us to recover the file and understand how the binary works
## Prerequisites
- Ghidra
- gdb
- A hex editor (hexeditor for linux, HxD for windows)
- Linux (prefferably)
## Recovering the file
First things first i wanted to know which type of file is it, we can do that by using the `file` command
![[Pasted image 20250313122634.png]]
We dont get much informations from data, so the first thing that came to mind from 'recovering the file' is the file header signature (declares the type of the file)

Standard ctf binaries are ELF files with the type .ELF, so by using a hex editor we can verify that with the `hexeditor` command
![[Pasted image 20250313123030.png]]

![[Pasted image 20250313123049.png]]
As expected, the header signature has been tampered with. We can retore it to it's original form by giving it it's correct hex header.

In my case i used GCK's table (https://www.garykessler.net/library/file_sigs.html)

By searching in the page for the .ELF extension we get the correct signature
![[Pasted image 20250313123322.png]]

![[Pasted image 20250313123358.png]]

Now by doing the `file` command again we should get something else

![[Pasted image 20250313123441.png]]
There it is! ELF pie executable, dynamically linked and not stripped. That's good news because non stripped binaries still have their debugging info built into them, so we can maybe use something like gdb or gcc -g in the future

But for now, let's run it
![[Pasted image 20250313123927.png]]
Don't forget to add executable rights
![[Pasted image 20250313124002.png]]
Nothing that can help us here

Next thing i did was to open up the binary inside ghidra to get some more info and maybe find the main function (Function you cant to look for in c code)
![[Pasted image 20250313124909.png]]
We create a new project, then file -> import file and choose our binary, we click ok and we get greeted by informations about our binary
![[Pasted image 20250313125218.png]]
Click ok and then double click on the file

A pop up will ask you if you want the file analyzed, click yes. This will give you the feature to see pseudo c code to understand the file more easily rather than assembly
![[Pasted image 20250313125253.png]]

Now you should be here

![[Pasted image 20250313125416.png]]

I went directly into the functions folder in the middle left pane and searched for the `main` function

![[Pasted image 20250313125527.png]]

Double click on it and here is it's pseudo-code

![[Pasted image 20250313125600.png]]

So basically when we run the executable it prints to the terminal "Running the program...", "Nothing to see here. Move along." and runs a mysterious `get_it()` function

Let's take a look at it
![[Pasted image 20250313125753.png]]

- In here we define:
	- An array abStack with the length of 60 
	- An unsigned integer that'll probably act as a loop counter like `i`

Then we have a for loop that iterates from when `local_c = 0` until `local_c = 0x38` (0x38 in decimal is 56, remember this for later on)

In each iteration we store inside the `abStack` array at the index `local_c` => `abStack[local_c]` or `abStack[i]`. We store a mysterious `encoded_flag[local_c]` XOR 0x2b (^ is XOR)

- So we can do 3 things here:
	- Search for the `abStack` array when binary runs and get it's value
	- Search for the `encoded_flag` array and XOR it with 0x2b in Cyberchef
	- Use the `strings` command to search for the position of either one of them

I went in with the 3rd option by using the `strings -tx onion | grep var`, this will output any strings that matches 'var' and it's memory address
![[Pasted image 20250313131012.png]]

We got nothing for the `abStack` array but we now know the encoded_flag is positionned at the address memory 0x3588, but we aren't going to find the encoded flag there cleanly because it's a combination of other instructions with it, not only that array

Now let's try the second option:

Take a look at the get_it() assembly code and you'll find the encoded_flag highlighted in pink
![[Pasted image 20250313134354.png]]

Double click it

![[Pasted image 20250313134445.png]]

Here is our encoded flag at each index of the array, let's first get these hex values more cleanly

Extracting the values using gdb
![[Pasted image 20250313135759.png]]

We know that the encoded flag is in the `.rodata` section by seeing it at the top left panel of ghidra

![[Pasted image 20250313142617.png]]

Use the command `info files` to get the starting and ending addreses of .rodata
![[Pasted image 20250313135920.png]]

Start address is 0x0000000000002000 for .rodata, but the the encoded flag doesn't start being used until the address 0x0000000000002000 as we see in ghidra

![[Pasted image 20250313140924.png]]

Remember the length of the encoded_flag array? 56 bytes

What we're tying to do now is to collect all the hex values from 0x0000000000002020 and advance for 56 bytes

We can do this using this command `x/56bx 0x0000000000002020`

![[Pasted image 20250313141139.png]]

Copy our hex values line by line into cyberchef

![[Pasted image 20250313141342.png]]

- What we want to do now is:
	- convert these values back to their original form
	- XOR them with 0x2b

So choose from the left dropdown menu the 'From Hex' and 'XOR' operations, put in the key '0x2b' inside the XOR operation. And voila!

![[Pasted image 20250313141633.png]]