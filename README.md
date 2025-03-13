# 0nion (!mario)
- Once we download the assets we get a .yaml file containing informations about the ctf and telling us to recover the file and understand how the binary works
## Prerequisites
- Ghidra
- gdb
- A hex editor (hexeditor for linux, HxD for windows)
- Linux (prefferably)
## Recovering the file
First things first i wanted to know which type of file is it, we can do that by using the `file` command

![file1](https://github.com/user-attachments/assets/b708a59f-440c-4d55-afcb-77206991aeb4)

We dont get much informations from data, so the first thing that came to mind from 'recovering the file' is the file header signature (declares the type of the file)

Standard ctf binaries are ELF files with the type .ELF, so by using a hex editor we can verify that with the `hexeditor` command

![hexeditor1](https://github.com/user-attachments/assets/95351753-3a71-4dc7-93a1-f35e3d34e959)

![hexeditor2](https://github.com/user-attachments/assets/7004ace1-7c12-4fbd-b232-c6d308781b79)

As expected, the header signature has been tampered with. We can retore it to it's original form by giving it it's correct hex header.

In my case i used GCK's table (https://www.garykessler.net/library/file_sigs.html)

By searching in the page for the .ELF extension we get the correct signature

![signatures](https://github.com/user-attachments/assets/60212fb4-9a45-4a9d-9283-12678799c737)

![hexeditor3](https://github.com/user-attachments/assets/79acf15d-3c55-40af-b6f2-586406864224)

Now by doing the `file` command again we should get something else

![file2](https://github.com/user-attachments/assets/2359daff-c879-4fb8-9af3-2d21819d74e2)

There it is! ELF pie executable, dynamically linked and not stripped. That's good news because non stripped binaries still have their debugging info built into them, so we can maybe use something like gdb or gcc -g in the future

But for now, let's run it

![exec1](https://github.com/user-attachments/assets/530756f1-a436-463a-a413-45db715a8f25)

Don't forget to add executable rights

![exec2](https://github.com/user-attachments/assets/8a46133a-e9e2-4f71-a268-234982b98842)

Nothing that can help us here

Next thing i did was to open up the binary inside ghidra to get some more info and maybe find the main function (Function you cant to look for in c code)

![ghidraRun](https://github.com/user-attachments/assets/a66860d3-910f-4b40-8ad7-3bbf465615c6)

We create a new project, then file -> import file and choose our binary, we click ok and we get greeted by informations about our binary

![ELFinfo](https://github.com/user-attachments/assets/ad83cfc0-9612-4a0e-aede-d6312487b39d)

Click ok and then double click on the file

A pop up will ask you if you want the file analyzed, click yes. This will give you the feature to see pseudo c code to understand the file more easily rather than assembly

![analyze](https://github.com/user-attachments/assets/66150eac-25b3-4c80-b0d6-d74ea8ae423b)

Now you should be here

![ghidraHome](https://github.com/user-attachments/assets/21fea637-75de-4be1-82ae-78d4b431977d)

I went directly into the functions folder in the middle left pane and searched for the `main` function

![midLeftPane](https://github.com/user-attachments/assets/77c9e89a-5eda-48b9-96a5-735ebe309414)

Double click on it and here is it's pseudo-code

![mainFunc](https://github.com/user-attachments/assets/25f93cf1-d4a8-44d7-9fbf-c1fd7648b071)

So basically when we run the executable it prints to the terminal "Running the program...", "Nothing to see here. Move along." and runs a mysterious `get_it()` function

Let's take a look at it

![getItFunc](https://github.com/user-attachments/assets/86e5a62a-f793-432e-995c-623154d4f92d)

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

![strings1](https://github.com/user-attachments/assets/7f4e00d4-b3c5-45f4-8ffb-5c30eb9915db)

We got nothing for the `abStack` array but we now know the encoded_flag is positionned at the address memory 0x3588, but we aren't going to find the encoded flag there cleanly because it's a combination of other instructions with it, not only that array

Now let's try the second option:

Take a look at the get_it() assembly code and you'll find the encoded_flag highlighted in pink

![assembly1](https://github.com/user-attachments/assets/93e1d283-4f5a-495d-b783-27825c4966b6)

Double click it

![assembly2](https://github.com/user-attachments/assets/589b5d95-1bce-4174-b231-96949fba8598)

Here is our encoded flag at each index of the array, let's first get these hex values more cleanly

Extracting the values using gdb

![gdb1](https://github.com/user-attachments/assets/c75bd139-5077-42d7-a62f-74501dd9e673)

We know that the encoded flag is in the `.rodata` section by seeing it at the top left panel of ghidra

![topLeftPane](https://github.com/user-attachments/assets/d68b4c97-6af5-4a6f-9cd7-9a7e230442b5)

Use the command `info files` to get the starting and ending addreses of .rodata

![infoFiles1](https://github.com/user-attachments/assets/22c6e1a3-9663-4dc1-bf23-469027c547b2)

Start address is 0x0000000000002000 for .rodata, but the the encoded flag doesn't start being used until the address 0x0000000000002020 as we see in ghidra

![assembly3](https://github.com/user-attachments/assets/6ddd5c3c-6597-4b2c-9142-de96377cac47)

Remember the length of the encoded_flag array? 56 bytes

What we're tying to do now is to collect all the hex values from 0x0000000000002020 and advance for 56 bytes

We can do this using this command `x/56bx 0x0000000000002020`

![cleanHex](https://github.com/user-attachments/assets/5ab2f6a6-b5de-480f-9e3b-884b7bd62f0d)

Copy our hex values line by line into cyberchef

![cyberchef1](https://github.com/user-attachments/assets/ec64812c-1102-47be-9cb0-2c9f00dfd97c)

- What we want to do now is:
	- convert these values back to their original form
	- XOR them with 0x2b

So choose from the left dropdown menu the 'From Hex' and 'XOR' operations, put in the key '0x2b' inside the XOR operation. And voila!

![cyberchef2](https://github.com/user-attachments/assets/2f54f63a-3ef6-4d69-9d35-6d9e7914cb6b)
