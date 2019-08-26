Title: Diving Into Ethereum Smart Contracts
Date: 2018-12-20
Author: Zach Faddis
Category: Blog

# Table of Contents

-   [Introduction](#orgf670121)
-   [The contracts](#org907f8c8)
    -   [Payment contract: Invoice](#orge31ced5)
    -   [Gambling contract: Bet](#org650da69)
        -   [initation](#orga53abb7)
        -   [confirmation](#org645f2e1)
        -   [judgment](#org58d68f9)
        -   [decision](#org4d21563)
        -   [source code](#org6976734)
    -   [Judge interface and contract: Simple Judge](#orgd6ac556)
        -   [Judge Interface](#org1d348ff)
        -   [Simple Judge](#org0420b9f)
-   [The Tools](#org31761bc)
    -   [The Platform: Ethereum](#org7c395c5)
    -   [The Contract Language: Solidity](#org1e929c2)
    -   [The Ethereum API: Web3.js](#org22475af)
    -   [The Framework: Truffle](#org663866f)
    -   [The Test Network: Ganache](#org9950fd7)

<a id="orgf670121"></a>

# Introduction

Lately I&rsquo;ve been diving into the world of Ethereum development and creating
some toy smart contracts in preparation of a much larger endeavor I have in
mind. 

I figured I&rsquo;d make a blog post showcasing the contracts I&rsquo;ve made and the
tools I used to create them. In the hopes of possibly getting others up to
speed faster and with less headaches in the future.


<a id="org907f8c8"></a>

# The contracts

Here I&rsquo;m going to describe the contracts I made and show the code for how they
are implemented. I&rsquo;ve also written tests to prove basic functionality that you
can see be looking at the source code in my [repo](https://github.com/Zacharius/eth_learning).

I am confident that these contracts work as intended but can&rsquo;t guarantee that
they are secure or can&rsquo;t be cheated in some way by a malicious actor. I think
they could be used for small values or in scenarios where you trust all
parties to act in good faith but I wouldn&rsquo;t use them for anything more serious
until they&rsquo;re looked at by someone with more experience. 


<a id="orge31ced5"></a>

## Payment contract: Invoice

My first foray into smart contracts is a simple invoice contract. It allows
someone, the Originator, the issue an invoice to another, the Debtor. 

The debtor can make query the contract to see how much he owes and make
partial payments until he pays the original amount in full. If he overpays he
can collect that money back as a refund. 

The Originator can collect however much the debtor has paid so far, until he
has collected the original amount.

    pragma solidity ^0.4.0;
    
    contract invoice {
    
      address public originator;
      address public payer;
      uint public og_owed;
      uint public owed;
      string public invoice_name;
    
      uint public refund = 0;
      uint public collected = 0;
    
      
    
      constructor(
    	      string _invoice_name,
    	      address _payer,
    	      uint _amount_owed
    	      )public {
        originator = msg.sender;
        payer = _payer;
        invoice_name = _invoice_name;
        owed = _amount_owed;
        og_owed = owed;
      }
    
      function pay() public payable {
    
        //payment must be from deptor
        require(
    	    msg.sender == payer,
    	    'This weight is not yours to bear');
    
    
        if(msg.value > owed) {
          refund = msg.value - owed;
          collected = owed;
          owed = 0;
        }else {
          collected += msg.value;
          owed -= msg.value;
        }
    
    
    
       }
    
    
      function collect() public {
        if (msg.sender == originator &&
    	collected > 0) {
          uint amount = collected;
          collected = 0;
          originator.transfer(amount);
        }
    
        if (msg.sender == payer &&
    	refund > 0) {
          amount = refund;
          refund = 0;
          payer.transfer(amount);
        }
      }
    
       
    }


<a id="org650da69"></a>

## Gambling contract: Bet

My most complicated contract by far, and the one I&rsquo;m most proud of. 

This is a contract to allow two people to bet on something and for
the outcome of the bet to be determined by a panel of predefined
judges. 


<a id="orga53abb7"></a>

### initation

the contract creator will specify a number of criteria to make the
bet.

-   who he is betting against
-   the amount being bet,
-   a short string defining the bet,
-   a list of judges,
-   a future date defining the deadline for judge&rsquo;s to cast their vote
-   the judge consensus threshold needed to determine a winner


<a id="org645f2e1"></a>

### confirmation

the person being challenged to the bet has 24 hours to send an amount
equal to the initiating bet. Once this has been done the bet will be
considered confirmed and neither party can retract their bet.  

The challenged can pay partially, allowing the payment to be split
into multiple chunks. If he overpays, he will be refunded the
difference. 	  

During the confirmation period, either party can choose to retract
their bet. In this event both parties will be refunded however much
they have put in so far and the contract will be considered complete. 


<a id="org58d68f9"></a>

### judgment

After the confirmation period, all judges listed will be able to vote
to determine the winner. Judges have 3 choices

-   challenged
-   challenger
-   hung

By default the judges&rsquo; vote will be set to hung, which means that
neither party should win. They have till the judgment deadline
determined in the contract initiation to cast their vote. A judge&rsquo;s
vote can be changed at any time before the deadline


<a id="org4d21563"></a>

### decision

At the judgment deadline all votes will be tallied. If their are
enough votes toward either party to cross the specified threshold, all
collected money will go towards the winning party. If neither party
gets enough votes, the bet will be considered hung and both parties
will be able to collect their money back. 


<a id="org6976734"></a>

### source code

    pragma solidity >= 0.5.0;
    
    
    contract Bet {
      address public initiator;
      address public challenged;
      address[] public judges;
    
      enum Vote { Null, Initiator, Challenged, Hung }
      mapping(address => Vote) public votes;
    
      string public bet_name;
    
      uint public bet;
      uint public matched = 0;
    
      //number between 0 and 100, inclusive
      //decides vote threshold needed to declare winner
      uint public threshold;
    
      uint public owed_initiator = 0;
      uint public owed_challenged = 0;
    
      //times expressed as seconds since unix epoch
      uint public creation_time = now;
      uint public judgement_deadline;
      uint public confirmation_deadline = creation_time + 1 days;
    
      event Voted(address judge, Vote vote);
      event stateChange(State state);
      event Owed(address owed, uint amount);
      event Test(uint t);
    
      enum State { Confirmation, Judgement, Complete }
      State public state;
    
      enum Conclusion { Undecided, Initiator, Challenger, Hung }
      Conclusion public winstate = Conclusion.Undecided;
    
      //check the time to ensure we are in the correct state
      modifier checkState {
        uint time = now;
    
        if (time > confirmation_deadline && state==State.Confirmation) {
          state = State.Judgement;
          emit stateChange(state);
        }
    
        if (time > judgement_deadline && state==State.Judgement) {
          state = State.Complete;
          emit stateChange(state);
        }
    
        _;
          
      }
    
      constructor (string memory _bet_name,
    	      address _challenged,
    	      address[] memory _judges,
    	      uint _threshold,
    	      uint _deadline) public payable {
    
        require(_deadline > confirmation_deadline,
    	    'deadline must be more than 24 hours into the future');
        
        require(_threshold <= 100,
    	    'threshold cant be greater than 100');
    
        require (_judges.length >= 1,
    	     'There must be atleast 1 judge');
        
        bet_name = _bet_name;
        bet = msg.value;
        challenged = _challenged;
        initiator = msg.sender;
        threshold = _threshold;
        judgement_deadline = _deadline;
        judges = _judges;
    
        for(uint i=0; i<_judges.length; i++) {
          votes[_judges[i]] = Vote.Hung;
        }
    
        state = State.Confirmation;
        emit stateChange(state);
      }
    
      //allow challenged to match bet of initiator
      function matchBet() public payable checkState {
    
        require(msg.sender == challenged,
    	    'you have not been challenged');
        require(state == State.Confirmation,
    	    'We are not in the confirmation stage');
    
        matched += msg.value;
    
        if(matched > bet) {
          owed_challenged = matched - bet;
          emit Owed(challenged, owed_challenged);
          matched = bet;
        }
    
        if(matched == bet) {
          state = State.Judgement;
          emit stateChange(state);
        }
      }
    
      //allow initiator and challenged to collect any owed amount
      function collect() public checkState {
        uint amount = 0;
    
        if(msg.sender == challenged &&
           owed_challenged > 0){
          amount = owed_challenged;
          owed_challenged = 0;
          msg.sender.transfer(amount);
        }
    
        if(msg.sender == initiator &&
           owed_initiator > 0){
          amount = owed_initiator;
          owed_initiator= 0;
          msg.sender.transfer(amount);
        }
      }
    
    
      //allow initiator or challenged to retract bet during confirmation
      function retract() public checkState {
        require(msg.sender == initiator || msg.sender == challenged,
    	    'you cannot retract this bet');
        require(state == State.Confirmation,
    	    'it is too late to turn back');
    
        state = State.Complete;
        winstate = Conclusion.Hung;
        emit stateChange(state);
    
        owed_initiator = bet;
        if(owed_initiator > 0){
          emit Owed(initiator, owed_initiator);
        }
    
        owed_challenged = matched;
        if(owed_challenged > 0){
          emit Owed(challenged, owed_challenged);
        }
      }
    
      //allow judges to vote
      function castVote(uint int_vote) public checkState {
        require(votes[msg.sender] != Vote.Null,
    	    'Only judges can vote');
        require(state == State.Judgement,
    	    'It is not yet the time of reckoning');
    
        Vote vote = Vote(int_vote);
    
        votes[msg.sender] = vote;
        emit Voted(msg.sender, vote);
      }
    
      function conclude() public checkState {
        require(state == State.Complete,
    	    'Your fate is still being decided on');
        require(winstate == Conclusion.Undecided,
    	    'This contract has concluded');
        
        uint init_votes = 0;
        uint chall_votes = 0;
    
        bool init_threshold = false;
        bool chall_threshold = false;
    
        for(uint i=0; i<judges.length; i++){
          if(votes[judges[i]] == Vote.Initiator) {
    	init_votes += 1;
          }
          else if(votes[judges[i]] == Vote.Challenged){
    	chall_votes += 1;
          }
        }
    
        emit Test(init_votes);
        emit Test(chall_votes);
    
        if( (init_votes/judges.length) > (threshold/100) ){
          init_threshold = true;
        }
    
        if( (chall_votes/judges.length) > (threshold/100) ){
          chall_threshold = true;
        }
    
        if(init_threshold != chall_threshold){
          if(init_threshold == true) {
    	winstate = Conclusion.Initiator;
    	owed_initiator += bet*2;
          }else if(chall_threshold == true) {
    	winstate = Conclusion.Challenger;
    	owed_challenged += bet*2;
          }
        }else{
          winstate = Conclusion.Hung;
          owed_initiator += bet;
          owed_challenged += bet;
        }
    
        state = State.Complete;
    
        emit Owed(initiator,owed_initiator);
        emit Owed(challenged, owed_challenged);
        emit stateChange(state);
    
      }
    
      //force change state, for testing purposes only
      //ensure is commentted out before deploying contract for realsies
      function changeState(State _state) public {
        state = _state;
      }
    
      
    
    }


<a id="orgd6ac556"></a>

## Judge interface and contract: Simple Judge

I wanted to make it easy to create robo judges to interact with my betting
contract so I made an extremely simple judge interface. To implement the
interface, all you need is a smart contract with a function, castVote(),
which accepts an address to send the vote to. This function should then make
its decision and send the vote to the address given.

I made a simple judge that implemented the judge interface. When it receives
a castVote request it will check the time and decide who it will side in
favor of depending on whether the time is even or odd. 


<a id="org1d348ff"></a>

### Judge Interface

    pragma solidity >= 0.5.0;
    
    import './bet.sol';
    
    interface Judge {
    
      enum Vote { Null, Initiator, Challenged, Hung }
    
      function castVote (Bet toContract) external returns (Vote vote);
    }   


<a id="org0420b9f"></a>

### Simple Judge

    pragma solidity 0.5.0;
    
    import './judge_interface.sol' ;
    import './bet.sol' ;
    import "./oraclizeAPI.sol";
    
    
    contract Simple_Judge is Judge {
    
    
      function castVote(Bet bet_contract) external returns (Vote vote) {
        if (block.timestamp % 2 == 0){
          vote = Vote.Initiator;
        }
        else {
          vote = Vote.Challenged;
        }
        bet_contract.castVote(uint(vote));
        return vote;
      }
    
    }


<a id="org31761bc"></a>

# The Tools

My goal here is to provide a roadmap of the quickest route to getting started
making things using Blockchain and Smart Contracts. Below is what I found the
be the most supported and well documented tools. Things change rapidly in this
space and I can only say that this is what I found to be the best tools at the
time of this writing.


<a id="org7c395c5"></a>

## The Platform: Ethereum

If you are new to the world of blockchain based coins and platforms you have
likely only heard of two technologies: Ethereum and Bitcoin. 

Bitcoin is strictly a currency. It has a very limited language that allows
for basic smart contract functionality, but it really isn&rsquo;t good for
anything more complicated than requiring multiple people&rsquo;s approval to
approve a transaction. It is the first real use of blockchain, is truly
revolutionary, and has many use cases, but if your a developer trying to
making something complex and interesting with Smart Contracts, I don&rsquo;t
recommend Bitcoin. 

Ethereum is a little different. It technically has a currency, Ether, but it
basically serves as just a medium of exchange to fuel its true purpose. That
being a global computer in which the code of every smart contract is stored
and executed on multiple computers across the network, ensuring a consensus is
reached before payments are sent. Imagine if all laws and contracts were
coded into this network, and all matters decided by the consensus of the
network instead of being adjudicated by lawyers and courts. That&rsquo;s basically
the promise of Ethereum. It supports much more complicated smart contracts
and allows developers to build some pretty interesting things.

There are several platforms and coins out there that allow you to make smart
contracts at various levels of complexity but Ethereum is by far the most
mature platform out there and allows the greatest freedom in what you can
build with their smart contracts.

[Ethereum Foundation](https://www.ethereum.org/)


<a id="org1e929c2"></a>

## The Contract Language: Solidity

Solidity is the main contract language for Ethereum Smart Contracts and
maintained by Ethereum. It was based off JavaScript and has many of the same
conventions and syntax. 

There is another Smart Contract language developed by Ethereum, Vyper. This
one is based off Python. From my understanding it is not nearly as well
developed or supported, at least at time of writing. 

[Solidity Docs](https://solidity.readthedocs.io/en/v0.5.1/index.html)


<a id="org22475af"></a>

## The Ethereum API: Web3.js

Web3.js is the JavaScript API used to interact with an Ethereum network.  

While the contracts themselves will be written in Solidity, you will need to
interact with them in another language. Web3.js allows you to create and
interact with smart contracts and to query information about the network.

There is also a Web3.py library for Python, but there is a larger community
around the JS library. Mainly because most people are trying to build
something to be used through a browser and for that you&rsquo;ll need JavaScript.

[Web3.js docs](https://web3js.readthedocs.io/en/1.0/index.html)


<a id="org663866f"></a>

## The Framework: Truffle

Truffle seems to be the most mature framework for developing smart contracts
and to have the largest community. Has tools to help you develop, test, and
deploy your contracts. Also has a library that acts as an abstraction layer
on top of Web3.js that makes it simpler to interact with your contracts.

It should be noted that Truffle is not maintained by Ethereum but it is an
open source project.

[Truffle Docs](https://truffleframework.com/docs/truffle/overview)


<a id="org9950fd7"></a>

## The Test Network: Ganache

Built by the same organization that maintains Truffle, Ganache gives you your
own private network that you solely control and allows you to test and
inspect how your contract will act without having to spend any gas or waiting
for transactions to be confirmed.

[Ganache Homepage/Download](https://truffleframework.com/ganache%20)

