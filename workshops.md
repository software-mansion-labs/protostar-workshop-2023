# Workshops: Protostar - smooth smart contract development experience
Time: 25 min (your coffee won't even get cold) <br>
Protostar docs: https://docs.swmansion.com/protostar/docs/tutorials/testing/fuzzing
## What is Protostar?
It is smart contract development toolchain for StarkNet. It helps you with:
- Managing dependencies
- Testing contracts
- StarkNet interactions

## Why to use Protostar
- fast, uses custom implementation of StarkNet to improve the preformance
- frictionless - easy installation, easy to use
- flexible - cheatcodes allow to test a lot of scenarios
- actively developed - developers are very responsive, we try to address community needs

## Workshops agenda
- How to create a project in Protostar
- How to test your contract using Protostar tesing toolbox
- How to build, declare & deploy your contract <br>

We assume basic knowledge of cairo, but we hope you sill can get something from this workshop if you don't know cairo at all (Protostar docs are actually not a bad place for StarkNet newcomers).

We will build small smart contract storing balance which can be increased and read. 

## Install Protostar
To install Protostar just run following one-liner in your terminal:
```bash
curl -L https://raw.githubusercontent.com/software-mansion/protostar/master/install.sh | bash
```
We do not support Windows at the moment :(
## Create a project
After you have protostar installed you can create a project. Run this command in your terminal
```bash
protostar init
```
You should be asked for your new project name
```bash
project directory name: workshops-project
13:42:35 [INFO] Execution time: 31.63 s
```
After that you can cd into your project
```bash
cd workshops-project
```

## New project overview
Your project roughly contains 
- `src` - this is where your contracts are located
- `tests` - here are your tests
- `pyproject.toml` - protostar configuration file

Protostar provides some template code, you can check if everything works. Run 
```bash
protostar test
```
You should see something like thi
```bash
13:44:43 [INFO] Collected 1 suite, and 2 test cases (0.047 s)
[PASS] tests/test_main.cairo test_increase_balance (time=0.09s, steps=127)
       range_check_builtin=1
[PASS] tests/test_main.cairo test_cannot_increase_balance_with_negative_value (time=0.01s)

13:44:46 [INFO] Test suites: 1 passed, 1 total
13:44:46 [INFO] Tests:       2 passed, 2 total
13:44:46 [INFO] Seed:        2906210099
13:44:46 [INFO] Execution time: 5.01 s
```
Cool, template tests pass!

## Contract
Below there is a simple contract we will be testing during this workshop. <br>
Contract includes one plain balance storage variables and two external methods to increase the balance and get current balance. Constructor sets its initial balance to 0 when contract is deployed.
```bash
%lang starknet
from starkware.cairo.common.math import assert_nn
from starkware.cairo.common.cairo_builtins import HashBuiltin

@storage_var
func balance() -> (res: felt) {
}

@external
func increase_balance{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(
    amount: felt
) {
    with_attr error_message("Amount must be positive. Got: {amount}.") {
        assert_nn(amount);
    }

    let (res) = balance.read();
    balance.write(res + amount);
    return ();
}

@view
func get_balance{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() -> (res: felt) {
    let (res) = balance.read();
    return (res,);
}

@constructor
func constructor{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() {
    balance.write(0);
    return ();
}
```

## Unit Testing
Ok, we can now test if this simple contract works as it supposed to do. Let's write some tests then!

Protostar main testing flow may be considered a bit non-intutive, but after you get used to it makes much sense, we promise.
Each test suite(one file) is a smart contract itself. If you import external methods from other contract to test contract, those are merged into one contract which tests have full access to, because tests are part of this contract.

```bash
%lang starknet
from starkware.cairo.common.math import assert_nn
from starkware.cairo.common.cairo_builtins import HashBuiltin
from src.main import increase_balance, balance

@external
func test_increase{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() {
    increase_balance(42);
    let (current) = balance.read();
    assert current = 42;
    return ();
}
```
Keep in mind protostar auto-removes constructors from test contracts. 
### Fuzzing

[https://docs.swmansion.com/protostar/docs/tutorials/testing/fuzzing](https://docs.swmansion.com/protostar/docs/tutorials/testing/fuzzing)

```bash
%lang starknet
from starkware.cairo.common.math import assert_nn
from starkware.cairo.common.cairo_builtins import HashBuiltin
from src.main import increase_balance, balance

@external
func setup_increase{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() {
    %{ given(amount = strategy.felts(rc_bound=True)) %}
    return ();
}

@external
func test_increase{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(amount: felt) {
    increase_balance(amount);
    let (current) = balance.read();
    assert current = amount;
    return ();
}
```

## Integration tests
Sometimes you want to test your contract in interaction with other contracts, or test the constructor code. Protostar makes it simple! You can use cheatcode to do that! What are cheatcodes? Cheatcodes are function provided by Protostar to modify StarkNet in a ways which is impossible in normal cairo.

We want to use `deploy_contract` cheatcode. It declares and deploys a contract on local StarkNet instance.
Below is an example of a test deploying our contract and increasing its balance by calling the contract.
```
%lang starknet
from starkware.cairo.common.cairo_builtins import (
    HashBuiltin,
    SignatureBuiltin,
)
from starkware.cairo.common.alloc import alloc

@contract_interface
namespace MainContract {
    func increase_balance(amount: felt) {
    }

    func get_balance() -> (res: felt) {
    }
}

@external
func test_voting{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}() {
    alloc_locals;
    // Deploy the contract
    local contract_address;
    %{
        ids.contract_address = deploy_contract('./src/main.cairo').contract_address
    %}

    let (result) = MainContract.get_balance(contract_address);
    assert result = 0;

    MainContract.increase_balance(contract_address, 155);
    let (result) = MainContract.get_balance(contract_address);
    assert result = 155;
    return ();
}
```

## Build, Declare & Deploy


Our contract is somewhat finished, wo we will build and deploy it on testnet. 
You will need an account on testnet to follow that. If you don't have it you can learn how to create one here
TODO

### Building
Simple as that!
```
protostar build
```
You should see
```
Building projects' contracts
Class hash for contract "main": 0x2a5de1b145e18dfeb31c7cd7ff403714ededf5f3fdf75f8b0ac96f2017541bc
15:51:13 [INFO] Execution time: 1.90 s
```
Your compiled contract is located in `build` directory.


### Declare


To declare your contract you just simply run
```
protostar declare --account-address ACCOUNT_ADDRESS build/main.json --private-key-path PRIVATE_KEY_PATH
```


### Deploy
To deploy your contract you just simply run
```
protostar deploy ./build/main.json --network alpha-goerli
```
TODO

## What next
You are interested in extending your knowledge of Protostar further? Great! Here are some directions you can explore rich protostar toolbox.
- learn how to leverage protostar config to improve your workflow
- dive into other cheatcodes
- learn how to write fuzz tests
- learn how to script your deploys
- learn how to optimize your contract with transaction profiler

# Future plans (next 3 months)
- cairo 1 support since end of February/ beggining of march
- simplified testing flow, test won't be contracts anymore
- speed ups!, rust-vm integration, improved caching