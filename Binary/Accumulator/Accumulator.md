# Accumulator - 50 points

I found this [program](./accumulator64) ([source](./accumulator.c)) that lets me add positive numbers to a variable, but it won't give me a flag unless that variable is negative! Can you help me out? Navigate to /problems/accumulator/ on the shell server to try your exploit out!

Hint:
How many bytes can an int store? How are positive and negative numbers represented in C?

### Solution
###### Writeup by asinggih

below is what the program does

```sh
❯ ./accumulator64
The accumulator currently has a value of 0.
Please enter a positive integer to add: 3
The accumulator currently has a value of 3.
Please enter a positive integer to add: 5
The accumulator currently has a value of 8.
Please enter a positive integer to add: 
...

```
So basically we need to give an input to the program, and it will sum our inputs. 

By looking at the source code provided, it can be seen that:

* For each iteration, our input is being stored in the variable called ```n```
* The ```n``` of each iteration will be accumulated inside the variable ```accumulator```
* Both ```n``` and ```accumulator``` are stored as ```int``` data types.
* In order to get the <b>flag</b> we have to break out of the loop, by having a negative ```accumulator``` value.
* However it won't let us insert a negative number as our input!


```C
#include <stdlib.h>
#include <stdio.h>

int main(){

	int accumulator = 0;
	int n;
	while (accumulator >= 0){
		printf("The accumulator currently has a value of %d.\n",accumulator);
		printf("Please enter a positive integer to add: ");

		if (scanf("%d",&n) != 1){
			printf("Error reading integer.\n");
		} else {
			if (n < 0){
				printf("You can't enter negatives!\n");
			} else {
				accumulator += n;
			}
		}
	}
	gid_t gid = getegid();
	setresgid(gid,gid,gid);
	
	printf("The accumulator has a value of %d. You win!\n", accumulator);
	system("/bin/cat flag");

}
```

After looking at the hint, and a bit of googling, I found a workaround for this problem. ```int``` data type in C is 4 bytes, which has a maximum decimal value of ```2,147,483,647``` instead of ```4,294,967,295```, due to the fact that ```int``` stores its value as [signed integers](https://en.wikipedia.org/wiki/Signed_number_representations) (left most bit acts as a flag between a positive or a negative value). 

Therefore, we can create an overflow in the ```accumulator``` variable, due to the fact that it cannot properly hold a value greater than ```2,147,483,647```. At my first attempt, I inserted the value ```2,147,483,648``` into the program. However it detects it as a negative value.

```sh
❯ ./test                
The accumulator currently has a value of 0.
Please enter a positive integer to add: 2147483648
You can't enter negatives!
The accumulator currently has a value of 0.
Please enter a positive integer to add: 
```

```2,147,483,648``` is equal to ```[1]111 1111 1111 1111 1111 1111 1111 1111``` but since C is using signed integers, it detects my input as ```-1```. 

What i needed was to input the maximum value without overflowing the variable. Hence I input ```2,147,483,647```, which translates to ```[0]111 1111 1111 1111 1111 1111 1111 1111``` and as expected, it accepted the number. Hence, the program prompted me for another integer, and i inserted ```1```. 

```sh
❯ ./accumulator64  
The accumulator currently has a value of 0.
Please enter a positive integer to add: 2147483647
The accumulator currently has a value of 2147483647.
Please enter a positive integer to add: 1
The accumulator has a value of -2147483648. You win!
actf{signed_ints_aint_safe}

```

## Flag
>actf{signed_ints_aint_safe}


