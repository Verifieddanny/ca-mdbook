# Cairo Academy: Understanding a Basic ERC20 Token Contract

This chapter delves into a fundamental ERC20 token contract implemented in Cairo. This contract demonstrates the core functionalities of a standard ERC20 token, providing a practical example for learning how to build fungible tokens on StarkNet.

## Purpose and Functionality

The provided Cairo code defines a basic ERC20 token contract. It implements essential functionalities such as:

- **Token Minting:** Creating new tokens and assigning them to an address.
- **Token Transfer:** Moving tokens between different addresses.
- **Allowance and Approval:** Allowing one address to transfer tokens on behalf of another.
- **Balance and Supply Tracking:** Maintaining records of user balances and the total token supply.
- **Metadata:** Providing basic token information like name, symbol, and decimals.
- **Withdrawal:** Allowing the contract owner to withdraw tokens from the contract.

This contract serves as a foundational example for understanding the mechanics of ERC20 tokens within the StarkNet ecosystem.

``` rust
use starknet::ContractAddress;
use starknet::storage::Map;
use starknet::{get_caller_address, get_contract_address};

#[starknet::interface]
trait IToken<TContractState> {
    fn mint(ref self: TContractState, address: ContractAddress);
    fn transfer(ref self: TContractState, address: ContractAddress, amount: u128);
    fn approval(ref self: TContractState, from: ContractAddress, to: ContractAddress, amount: u128);
    fn allowance(self: @TContractState, from: ContractAddress, to: ContractAddress) -> u128;
    fn transfer_from(ref self: TContractState, from: ContractAddress, to: ContractAddress, amount: u128);
    fn withdrawTokens(ref self: TContractState, contract_address: ContractAddress, amount: u128);
    fn get_name(self: @TContractState) -> felt252;
    fn get_symbol(self: @TContractState) -> felt252;
    fn get_decimal(self: @ContractState) -> u128;
    fn get_total_supply(self: @ContractState) -> u128;
    fn get_balance_of_user(self: @TContractState, user: ContractAddress) -> u128;
    fn get_owner(self: @ContractState) -> ContractAddress;
}

#[starknet::contract]
mod ERC20Token {
    use super::IToken;
    use starknet::{ContractAddress, get_caller_address, get_contract_address};
    use starknet::storage::Map;

    #[storage]
    struct Storage {
        name: felt252,
        symbol: felt252,
        decimal: u128,
        total_supply: u128,
        owner: ContractAddress,
        balance_of: Map::<ContractAddress, u128>,
        allowance: Map::<(ContractAddress, ContractAddress), u128>,
    }

    #[constructor]
    fn constructor(ref self: ContractState) {
        self.name.write('ERC20Token');
        self.symbol.write('ETK');
        self.decimal.write(18);
        self.owner.write(get_caller_address());
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        TransferFrom: TransferFrom,
        Transfer: Transfer,
        Mint: Mint,
        Withdraw: Withdraw,
        Approval: Approval,
    }

    #[derive(Drop, starknet::Event)]
    struct TransferFrom {
        #[key]
        from: ContractAddress,
        to: ContractAddress,
        amount: u128,
    }

    #[derive(Drop, starknet::Event)]
    struct Transfer {
        #[key]
        to: ContractAddress,
        amount: u128,
    }

    #[derive(Drop, starknet::Event)]
    struct Mint {
        #[key]
        to: ContractAddress,
        amount: u128,
    }

    #[derive(Drop, starknet::Event)]
    struct Withdraw {
        #[key]
        contract_address: ContractAddress,
        user: ContractAddress,
        amount: u128,
    }

    #[derive(Drop, starknet::Event)]
    struct Approval {
        #[key]
        user: ContractAddress,
        to: ContractAddress,
        amount: u128,
    }

    #[abi(embed_v0)]
    impl ITokenImpl of super::IToken<ContractState> {
        fn mint(ref self: ContractState, address: ContractAddress) {
            let caller: ContractAddress = get_caller_address();
            assert(!caller.is_zero(), 'Caller cannot be address zero');
            let supply: u128 = self.total_supply.read();
            let balance: u128 = self.balance_of.read(get_caller_address());
            self.total_supply.write(supply + 1000);
            self.balance_of.write(get_caller_address(), balance + 1000);
            self.emit(Mint { to: get_caller_address(), amount: 1000 });
        }
        fn transfer(ref self: ContractState, address: ContractAddress, amount: u128) {
            let sender_balance: u128 = self.balance_of.read(get_caller_address());
            let reciever_balance: u128 = self.balance_of.read(address);
            assert(sender_balance >= amount, 'Not Enough Tokens');
            self.balance_of.write(get_caller_address(), sender_balance - amount);
            self.balance_of.write(address, reciever_balance + amount);
            self.emit(Transfer { to: address, amount: amount });
        }
        fn approval(ref self: ContractState, from: ContractAddress, to: ContractAddress, amount: u128) {
            self.allowance.write((from, to), self.allowance.read((from, to)) + amount);
            self.emit(Approval { user: from, to: to, amount: amount });
        }
        fn allowance(self: @ContractState, from: ContractAddress, to: ContractAddress) -> u128 {
            self.allowance.read((from, to))
        }
        fn transfer_from(ref self: ContractState, from: ContractAddress, to: ContractAddress, amount: u128) {
            assert(self.allowance.read((from, to)) >= amount, 'Insufficient Allowance');
            self.allowance.write((from, to), self.allowance.read((from, to)) - amount);
            assert(self.balance_of.read(from) >= amount, 'Not Enough Tokens');
            self.balance_of.write(from, self.balance_of.read(from) - amount);
            self.balance_of.write(to, self.balance_of.read(to) + amount);
            self.emit(TransferFrom { from: from, to: to, amount: amount });
        }
        fn withdrawTokens(ref self: ContractState, contract_address: ContractAddress, amount: u128) {
            let contract_balance = self.balance_of.read(get_contract_address());
            let caller_balance = self.balance_of.read(get_caller_address());
            assert(contract_balance >= amount, 'Contract balance Insufficient');
            self.balance_of.write(get_caller_address(), caller_balance + amount);
            self.balance_of.write(get_contract_address(), contract_balance - amount);
            self.emit(Withdraw { contract_address: contract_address, user: get_caller_address(), amount });
        }
        fn get_name(self: @ContractState) -> felt252 {
            self.name.read()
        }
        fn get_symbol(self: @ContractState) -> felt252 {
            self.symbol.read()
        }
        fn get_decimal(self: @ContractState) -> u128 {
            self.decimal.read()
        }
        fn get_total_supply(self: @ContractState) -> u128 {
            self.total_supply.read()
        }
        fn get_balance_of_user(self: @ContractState, user: ContractAddress) -> u128 {
            self.balance_of.read(user)
        }
        fn get_owner(self: @ContractState) -> ContractAddress {
            self.owner.read()
        }
```
