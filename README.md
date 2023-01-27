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


