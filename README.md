# Deep-Dive-Into-Solidity-Libraries
Solidity allows us to write libraries so we can reuse the same code to execute a certain computation. In this article we will look at how the EVM interprets libraries, and walk through an example of how to create one.


## What is a Library?
Libraries are similar to smart contracts, but, typically, are only deployed once. Their code is reused by taking advantage of ```delegatecall()```. This means that the context of the code is executed in the calling contract. The library functions will not be explicitly visible in the inheritance hierarchy. The use of ```delegatecall()``` can allow us to access state variables from our calling smart contract. However, since libraries are an isolated piece of code, you must supply the state variables for the library to access them.

Let’s look at an example:
```
pragma solidity^0.8.17;

library Lib {

    // sets the storgae variable to a new value
    function changeState(uint256[1] storage changeMe) public {
        changeMe[0] = 1;
    }

}

contract C {

    // stateVariable[0] initialized to 0
    uint256[1] stateVariable;

    function callChangeState() external returns (uint256[1] memory, uint256[1] memory){

        // stores state variable to memory so we can return it later
        uint256[1] memory _oldState = stateVariable;

        // calls our library function
        Lib.changeState(stateVariable);

        // stores our new state variable to memory so we can return it
        uint256[1] memory _newState = stateVariable;

        return (_oldState, _newState);

    }

}
```
Here is our output: <br>
```_oldState[0]```: 0 <br>
```_newState[0]```: 1 <br>

Perfect! This is what we were expecting to see. The storage variable we passed in is set to a different value via ```delegatecall()```. It is worth noting that when we call a library with a storage parameter, we are passing in the storage address (pointer to storage variable).

If you are calling a library function that is ```view``` or ```pure``` you can label that function as ```internal``` instead of ```public```. Doing so does not use a ```delegatecall()``` to call the function. Instead, the code is included in the calling contract. That means we call the function via a ```JUMP``` instruction. The ```JUMP``` instruction will only cost us 8 gas, while a ```delegatecall()``` can cost a varying amount of gas. ```delegatecall()```’s formula for gas costs is: ```gas_cost = access_cost + memory_expansion_cost + gas_sent_with_call```. ```delegatecall()``` is going to be more expensive, so I personally would recommend using ```internal``` over ```public``` when possible. However, there are some cases where a ```public``` function is a better fit. We will go over this later on in the article. One last thing worth noting when using an ```internal``` function is that the data is passed to the function by reference, instead of copying the data.

Since the library’s code is stored separately from your contract, you can obtain the address with the following code:
```
function getLibraryAddress() external view returns(address) {
        return address(Lib);
    }
```
Additionally, on deployment, you will need to deploy your library and specify the address to your smart contract. Here is how we would do that.
```
    const Lib = await ethers.getContractFactory("Lib");
    const lib = await Lib.deploy();
    await lib.deployed();

    const YourContract = await ethers.getContractFactory("YourContract", {
      libraries: {
        Lib: lib.address
      },
    });
    const yourContract = await YourContract.deploy();
    await yourContract.deployed();
```

Before moving on, let’s look at some key differences between libraries and smart contracts. <br>
- There are no state variables in libraries. <br>
- Inheritance is not allowed with libraries. <br>
- Libraries cannot receive Ether. <br>
- Libraries cannot be destroyed <br>


## Function Signatures & Selectors
Libraries do not use the conventional smart contract ABI when they are called. Although it is similar, there are some key differences. The main difference is that libraries can take storage variables and recursive structs as parameters. With a struct you need specify the fully qualified name of the struct (i.e. ```Contract Test { struct S {} }``` would be ```Test.S```). Mappings follow this format ```mapping(<keyType> => <valueType>) storage```. Notice that there is a space in between the parentheses and storage. For all other storage types, you use the same identifier as you would for a non-storage type, but you add a single space and “storage” to it. For all other variables you use the identifier you would use when getting a function signature for a smart contract.

The easiest way to get a function signature is using the following code:
```
function getSignature() external pure returns(bytes4) {
       return Lib.changeState.selector;
}
```


## Using For
A nice feature of libraries is ```using x for y```. This allows you to attach type y to function x. The first parameter must match the type you are attempting to attach it to or else the compiler will throw an error. You can specify multiple functions for a type like so, ```using { Lib.func1, Lib.func2 } for uint256```. Additionally, you can specify an entire library for all types with the following syntax ```using Lib for *``` This does not allow you to call any function on any type. The type of the first parameter still has to match the type of the variable you are using. This just makes it easier for you, so you don't have to keep using ```using x for y```. 
Let’s take a look at some examples. Add the following function to our library.
``` 
function square(uint256 _val) public pure returns (uint256){
   return _val * _val;
}
```
Now in our contract C, we will add this piece of code.
```
using { Lib.square }  for uint256;

function testUsing() external returns(uint256) {
   return uint256(5).square();
}
```
Nothing crazy here, our output will be 25 as expected. Let’s see what happens if we try to assign our library to all variables.
```
using Lib  for *;

function testUsing() external returns(uint256) {

    // calls square() on 5
    uint256 callsSquare = uint256(5).square();

    // changes our storage variable to 1
    stateVariable.changeState();

    // throws an error
    stateVariable.square(); // comment out

    return  stateVariable[0] + callsSquare;
}
```
If you try to run this function as is, you will get an error saying ```square()``` is not visible. This is because we are attaching ```square()``` to a variable that is not a ```uint256```. Ok, let’s comment out that line now and try running it again. Great, 26 is our output! We get it by squaring 5 (25), then changing our state variable to 1 and summing both of them.

Our last topic for this section will be covering using ```global```. ```global``` allows us to attach libraries to custom data types (like structs) even if the contracts are located in different files. Let’s look at an example. First say we have file ```globalTest1.sol```:
```
pragma solidity^0.8.17;

// our custom data type
struct DataType {
    uint256 var1;
    uint256 var2;
}

// our library that performs an operation on our custom data type
library Lib {

    function sumDataType(DataType memory _data) internal pure returns (uint256) {
        return _data.var1 + _data.var2;
    }

}

// attaching our library to this custom data type
using { Lib.sumDataType } for DataType global;

contract C1 {
    // simple function inside our file that demonstrates it works
    function getResult() external view returns(uint256) {
        DataType memory data = DataType(1,2);
        return data.sumDataType();
    }
}

```
Also let's say we have a separate file called ```globalTest2.sol``` that looks like this.
```
pragma solidity^0.8.17;

// import ONLY our custom data type
import { DataType } from "./globalTest1.sol";

contract C2 {
    // simple function to verify it works properly
    function getResult() external view returns(uint256) {
        DataType memory data = DataType(3,4);
        return data.sumDataType();
    }
}
```
If we call ```getResult()``` in both contract we will get the following output: <br>
```C1.getResult()```: 3 <br>
```C2.getResult()```: 7 <br>

Notice we are ONLY importing our custom data type from ```globalTest1.sol```. However, we still get to use our library because we attached it globally.

This wraps up the section on ```use for```. Next we will begin to look at the process of creating our own library!

## Writing a Library
To make sure I had a good grasp of libraries before writing this article, I decided to write and publish my first library. In this section, we will go over that library, some design choices I made based on what I learned from writing my first library, and how to publish it as a npm package.

<br>
Here is a link to the Github repository: https://github.com/marjon-call/memLibrary <br>
This library is heavily written with Yul. Although I will describe what the code is doing, if you would like to get a better understanding of Yul here is a link to an article I published on it: https://medium.com/coinsbench/beginners-guide-to-yul-12a0a18095ef

### Library Overview
If you have ever worked with memory arrays before in Solidity I am sure you have encountered that you can not use ```push()``` with them. Although not explicitly defined as fixed sized arrays, memory arrays do not allow developers to perform the same operations as dynamic arrays in storage. This library tackles that issue by allowing users to ```push()```, ```pop()```, ```insert()```, and ```remove()``` elements in memory arrays. This library supports the following types: ```uint8[]``` - ```uint256[]```, ```int8[]``` - ```int256[]```, ```address[]```, ```bool[]```, & ```bytes32[]```. For this tutorial, we will only be going over the ```uint256``` type, but all types work the same. For instance ```myAddressArray.push(myAddress)``` and ```myUint256Array.push(myUint256)``` would both work. The reason we are allowed to have the same name for both types is because the function signature look differently so there are no collisions (i.e. ```push(address[], address)``` vs ```push(uint256[],uint256)```. 

Let’s dive into the code now!

### Push()
```push(uint256[] memory arr, uint256 newVal)``` takes in as the parameters the memory array we want to manipulate and the value we want to append to the end of the array. It returns an array that we store to the old array in our code.
Here is the example we will use for ```push()```:
```
function testPush(uint256[] memory arr) external pure returns(uint256[] memory) {
    arr = mem.push(arr, 4);
    return arr;
}
```
I will be passing in the array ```[0, 1, 2, 3]```, and we will see it returns ```[0,1,2,3,4]```.
Now let’s look at the code!
```
// allows user to push value to memory array
    function push(uint256[] memory arr, uint256 newVal) public pure returns(uint256[] memory) {

        assembly {
   
            // where array is stored in memory
            let location := arr
   
            // length of array is stored at arr
            let length := mload(arr)
   
            // gets next available memory location
            let nextMemoryLocation := add( add( location, 0x20 ), mul( length, 0x20 ) )

            let freeMem := mload(0x40)

            // advance msize()
            let newMsize := add( freeMem, 0x20 )

            // checks if additional varibales in memory
            if iszero( eq( freeMem, nextMemoryLocation) ){

                let currVal
                let prevVal
               
                // makes room for _newVal by advacning other memory variables locations by 0x20 (32 bytes)
                for { let i := nextMemoryLocation } lt(i, newMsize) { i := add(i, 0x20) } {
                   
                    currVal := mload(i)
                    mstore(i, prevVal)
                    prevVal := currVal
                   
                }

            }

            // stores new value to memory
            mstore(nextMemoryLocation, newVal)
   
            // increment length by 1
            length := add( length, 1 )
   
            // store new length value
            mstore( location, length )
   
            // update free memory pointer
            mstore(0x40, newMsize )
   
        }

        return arr;
        
    }
```
The first thing that happens in this function is we get where, in memory, our array is located. In this case it will be located at memory location ```0x80```. Next we get the length of the array by loading the location from memory. That will return 4. So we know now the next four memory locations will be where our array is stored in memory. However, we need an equation that works despite the size of the array. We need to take our location and add ```0x20``` (this advances us to element 0 of our array by adding 32 bytes). Next we multiply the length of our array (what is being stored in ```0x80``` by 32 bytes). Then we sum these two numbers. Now we store the next memory location for our array to ```nextMemoryLocation```. Now we need to find out if there are any other variables in memory. We do this because we don’t want to overwrite any other variables. This is done by loading the free memory pointer (```0x40```). Since we plan on adding a new variable to memory, we also want to create a new stack variable that keeps track of our free memory pointer after we add a new variable to memory (```newMsize```). Now we check if the free memory pointer is equal to our next memory location for our array. If it is, we do not have to worry about overwriting any memory variables. In our example we do not have to worry about this. In the case that we did have other memory variables, we loop through our memory locations and store each variable in the following memory location. Now, we can store our new memory variable. Then we have to update the length of our array and store that. Finally, we need to update our free memory pointer so Solidity knows not to overwrite our last stored memory location. Now we can return our new array.


Now let’s look at our memory layout at the end of our function call.
|   Memory Location |  Value  |
|  :---:   | :--- |
|   0x00  |   Empty  |
|   0x20  |   Empty  |
|   0x40  |  ```0x2c0``` (Free memory Pointer) |
|   0x60  |   Empty  |
|   0x80  |   4 (size of initial memory array)  |
|   0xa0  |   0 (```arr[0]```)  |
|   0xc0  |   1 (```arr[1]```)  |
|   0xe0  |   2 (```arr[2]```)  |
|   0x100  |   3 (```arr[3]```)  |
|   0x120  |   ```0x20``` (Offset, tells us to step over the size of array)  |
|   0x140  |   5 (new size of array)  |
|   0x160  |   0 (```arr[0]```)  |
|   0x180  |   1 (```arr[1]```)  |
|   0x1a0  |   2 (```arr[2]```)  |
|   0x1c0  |   3 (```arr[3]```)  |
|   0x1e0  |   4 (```arr[4]```) [At this point we are returning from library call] |
|   0x200  |    5 (new size of array)   |
|   0x220  |   0 (```arr[0]```)  |
|   0x240  |   1 (```arr[1]```)  |
|   0x260  |   2 (```arr[2]```)  |
|   0x280  |   3 (```arr[3]```)  |
|   0x2a0  |   4 (```arr[4]```)  [We are storing array in ```testPush()```] |
|   0x2c0  |   ```0x20``` (Offset, tells us to step over the size of array) |
|   0x2e0  |   5 (new size of array)  |
|   0x300  |   0 (```arr[0]```)  |
|   0x320  |   1 (```arr[1]```)  |
|   0x340  |   2 (```arr[2]```)  |
|   0x360  |   3 (```arr[3]```)  |
|   0x380  |   4 (```arr[4]```) [We are returning array from ```testPush()```] |

There is a lot going on in memory right now, let’s go over what's happening. We store our original array to memory. Then, we need to call our library via ```delegatecall()```. There, we format our new array and return it. Remember ```return``` uses memory, and since we are using ```delegatecall()``` the context of the memory is in our calling contract. We now store our new array in ```testPush()```’s memory. Finally, we return our new array, which again uses memory. 

So, obviously, we are using a lot of memory. What would happen if we change our ```push()``` from ```public``` to ```internal```? 

Try it out and this time call our library like this:
```
mem.push(arr, 4);
```
  Let’s go over the memory in this instance.

|   Memory Location |  Value  |
|  :---:   | :--- |
|   0x00  |   Empty  |
|   0x20  |   Empty  |
|   0x40  |  ```0x140``` (Free memory Pointer) |
|   0x60  |   Empty  |
|   0x80  |   4 (size of new memory array)  |
|   0xa0  |   0 (```arr[0]```)  |
|   0xc0  |   1 (```arr[1]```)  |
|   0xe0  |   2 (```arr[2]```)  |
|   0x100  |   3 (```arr[3]```)  |
|   0x120  |   4 (```arr[4]```)  |
|   0x140  |   ```0x20``` (Offset, tells us to step over the size of array)  |
|   0x160  |   5 (new size of array) |
|   0x180  |   0 (```arr[0]```)  |
|   0x1a0  |   1 (```arr[1]```)  |
|   0x1c0  |   2 (```arr[2]```)  |
|   0x1e0  |   3 (```arr[3]```)  |
|   0x200  |   4 (```arr[4]```)  [We are returning array from ```testPush()```]  |

We just cut down on so much memory! Because we do not need to store our new array variable or return as much data. Clearly this is the better option, right? Well actually this might work better in the case that we only have one memory variable, but let’s change our function to add another memory variable and see what happens. 
Here is our new ```testPush()```:
```
function testPush(uint256[] memory arr) external pure returns(uint256[] memory, uint256[] memory) {
    uint256[] memory arr2 = new uint256[](1);
    arr2[0] = 100;
    mem.push(arr, 4);
    return (arr, arr2);
}
```
In this example we store another array in memory. It will be stored after the array we want to manipulate, because the array we manipulate is allocated to memory first from our parameters. We then return both arrays. Let’s check out the output:
```arr```: ```[0,1,2,3,4]```
```arr2```:  ```[1, 100, 64, 256]```
So ```arr``` looks great! But ```arr2``` looks a little off. We were expecting ```[100]```. Why does this happen? Internal calls do not execute in the context of our calling function. So when we tell our function to return ```arr2``` it looks at the previously defined memory location for it. We just overwrote that location to be the last element of ```arr``` (4). So now it is returning an array of size 4 for ```arr2```. Even though ```internal``` functions saves on memory allocation, it does not provide the functionality we desire, so we must use ```public``` functions for our library. With that in mind, let’s look at the rest of our functions for our library.

### Pop()

Similar to ```push()```, pop manipulates the end of our array. However, instead of appending an element, we will remove the last element of the array. Let’s take a look at our code.
```
// removes last element from array
function pop(uint256[] memory arr) public pure returns (uint256[] memory){

    assembly {

        // where array is stored in memory
        let location := arr

        // length of array is stored at arr
        let length := mload(arr)

        let freeMemPntr := mload(0x40)

        let targetLocation := add( add( location, 0x20 ), mul( length, 0x20 ) )

        for { let i := targetLocation } lt( i, freeMemPntr ) { i := add( i, 0x20 )} {
            // stores next vlaue in memory to current memory location
            let nextVal := mload( add(i, 0x20 ) )
            mstore(i, nextVal)

        }
        
        // update & store new length
        length := sub( length, 1 )
        mstore(location, length)
    }

    return arr;

}
```
Like last time, we get the location and length of our array. We also get the next available free space in memory. Then, we calculate the position of the element that needs to be removed. Next, we loop starting from our element that needs to be removed to our last position in memory. In the loop we store the succeeding value in our current memory location. Finally, we update length and store it before returning our new array.

### Insert()
```insert()``` allows us to store a new value at a specified index in the array. Here is the code.
```
// inserts element into array at index
function insert(uint256[] memory arr, uint256 newVal, uint256 index) public pure returns(uint256[] memory) {

    assembly {

        // where array is stored in memory
        let location := arr

        // length of array is stored at arr
        let length := mload(arr)

        // gets next available memory location
        let nextMemoryLocation := add( add( location, 0x20 ), mul( length, 0x20 ) )
       
        // fre memory pointer
        let freeMem := mload(0x40)

        // advance msize()
        let newMsize := add( freeMem, 0x20 )
        
        // location we want to insert element
        let targetLocation := add( add( location, 0x20 ), mul( index, 0x20 ) )
       
        let currVal
        let prevVal

        for { let i := targetLocation } lt( i, newMsize ) { i := add( i, 0x20 )} {

            currVal := mload(i)
            mstore(i, prevVal)
            prevVal := currVal

        }

         // stores new value to memory
        mstore(targetLocation, newVal)

        // increment length by 1
        length := add( length, 1 )

        // store new length value
        mstore( location, length )

        // update free memory pointer
        mstore(0x40, newMsize )

    }

    return arr;
    
}
```
Exactly like ```push()``` we get ```location```, ```length```, ```nextMemoryLocation```, ```freeMem```, and ```newMsize```. Like with ```pop()```, we get ```targetLocation```. Then starting from ```targetLocation```, we loop to ```newMsize``` replacing each value with the preceding value. After we store our new value, update the length of the array, and update the free memory pointer we can return our new array.

### remove()
Our last function for our library is ```remove()```. It allows us to remove an element from our array at a specified index. Here is the code for ```remove()```.
```
// removes element from array at index
function remove(uint256[] memory arr, uint256 index) public pure returns (uint256[] memory){

    assembly {

        // where array is stored in memory
        let location := arr

        // length of array is stored at arr
        let length := mload(arr)

    	// free memory pointer
        let freeMemPntr := mload(0x40)

	     // location of element being removed
        let targetLocation := add( add( location, 0x20 ), mul( index, 0x20 ) )

        for { let i := targetLocation } lt( i, freeMemPntr ) { i := add( i, 0x20 )} {

            let nextVal := mload( add(i, 0x20 ) )
            mstore(i, nextVal)

        }

        length := sub( length, 1 )

        mstore(location, length)
        
    }

    return arr;

}
```

Yet again we get ```location```, ```length```, ```freeMemPntr```, and ```targetLocation```. Now we loop through our array starting from ```targetLocation``` to  ```freeMemPntr``` and store the preceding value in our current memory location. Finally, we can update and return our new array.

### Publishing Our Library
Now that we finished writing our Library, we can publish our work with npm. The first step is to make sure we are in the folder with our code in the command line. Next, if we haven't done so yet, we need to create a ```package.json```. That can be done with the following command. <br>
```npm init -y``` <br>
Now that we have our ```package.json```. We need to either login or create an npm account with the following command. <br>
```npm login``` <br>
Finally we can publish our library with: <br>
```npm publish``` <br>

Congratulations! You now know how to create and publish libraries with Solidity!

This wraps up the article on Solidity libraries.

Here is the Solidity documentation on libraries: https://docs.soliditylang.org/en/v0.8.17/contracts.html#libraries

If you have any questions, or would like to see me make a tutorial on a different topic please leave a comment below.
If you would like to support me making tutorials, here is my Ethereum address: 0xD5FC495fC6C0FF327c1E4e3Bccc4B5987e256794.
