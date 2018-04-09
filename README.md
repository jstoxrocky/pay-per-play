# A Simple Pay-Per-View Pattern for Ethereum-Connected Web Applications

##### Disclaimer
*This article is not a DIY on how to implement a fully decentalized and trustless pay-per-view mechanism inside a smart-contract. It proposes a simple, light-weight pattern to enable pay-per-view purchases from a **centralized web application** without requiring a database, by moving value transfers and data storage to an Ethereum smart contract.*

<img src="https://images.unsplash.com/photo-1520239432281-c4011f9f46ef?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=0878477cb12832504383bdb4135051b7&auto=format&fit=crop&w=1500&q=80" />

##### Motivation
Value transfer over the internet is (rightly?) difficult for the hobbiest programmer to implement. Those wishing to build web applications dealing with payments have a few trustworthy but limited options at their disposal including Stripe, Paypal or Venmo APIs. These options, however, are fraught with ToS agreements, fees, taxes, expiring tokens, and limits on what what the API can do. This article proposes a lightweight solution that does not require a database by making use of Ethereum smart-contracts for value transfer and data storage. Specifically this article looks to provide a pattern useful for pay-per-view type payments in which a new payment is required for each repeated request for content. 

##### Existing Payment Patterns with Ethereum
Existing payment patterns used by Ethereum-connected web applications provide a level of security for server side payment validation, but fail when it comes to the repeated nature of pay-per-view type payments. Existing patterns mainly focus on payment validation, and address authentication. 

###### Payment Validation
One option available to web applications looking to validate payments is by parsing data included in Ethereum transactions. Users submit a transaction hash to a server and the server verifies that the transaction has been mined, contains the correct amount of ETH, and was sent to the correct address etc. (see [ethermango](https://www.reddit.com/r/ethdev/comments/85upbo/i_got_tired_of_joke_dapps_so_i_tried_making/dw0hhbl/)). 

An alternative option for payment validation requires users to transact to a certain smart-contract function, which when executed successfully, updates a Solidity mapping, marking the address as "paid" (see [ethbird](https://etherscan.io/address/0x0e57dd440090561c3c72df021dcc3d3bab3da5ab)). When a user requests some content, the sever will make a call to the contract to verify whether the user is in the "paid" bucket.

###### Address Authentication
Address authentication is used to prevent a user from impersonating an address that they do not control. Without address authentication checks in place, a user could grab another user's valid transaction hash and submit it to a web application to receive the content. For this reason, applications like [ethermango](https://ethermango.com/) require users to sign a randomly generated value and submit the signature to the server in tandem with the transaction hash. This allows the server to recover the signer address and verify that it matches the transaction sender.

###### Some Existing Examples

Web applications like Ethermango parse transaction data along with recovering addresses from signatures to secure payments. This prevents other users from impersonating a existing purchaser but it does not prevent an existing purchaser from receiving the content as many times as they like (think music streaming, or arcade game plays). Ethermango could solve this issue by storing transaction hashes in a database preventing them from being used more than once. 

Ethbird prompts users to execute a contract function to update a Solidity mapping, but it does not perform address authentication. This provides assurance that a good natured purchaser has truly paid, but does not prevent address impersonation, and it does not prevent a user from receiving the content as many times as they like. Ethbird could similarly be tweaked to perform address authentication and a database could potentially prevent purchasers from receiving their content multiple times.

For the hobbiest programmer, however, databases may not be a viable option since they can introduce large maintenance costs on hosting services like AWS.

##### Simple, Database Free, Pay-Per-View Payments  
This article's proposed pattern combines smart-contract logic to validate payments, smart-contract storage of payment information, and server-side address authentication.


###### 5 Steps to Success
<ol>
<li>A user signals to the web application's server that they wish to consume a piece of content (read an article, stream a track, or play a new game).
</li>

<li>The web application's server generates a random 32-byte value and sends it to the user.
</li>
 
<li>
The web application prompts the user execute the function `pay` on a smart contract such as the following:
    
    contract Payments {
        uint256 public price;
	    mapping (address => bytes32) public nonces;

	    function Payments(){
            price = 1000000000000000; // 0.001 ETH
        }
 
        function pay(bytes32 nonce) public payable {
            require(msg.value == price);
            nonces[msg.sender] = nonce;
        }
    
        function getNonce(address addr) public view returns (bytes32) {
            return nonces[addr];
        }
    }
This function can only execute if and only if the user submits the correct payment to the function and submits a value for the nonce argument. Upon successful execution of the function, the senders address is associated with whatever nonce value was submitted.
</li>
<li>
The web application prompts the user to sign the nonce value and send it back to the server
</li>
<li>
The server recovers the user's address from the signature and calls the contract's `getNonce` function to verify that the nonce from step 1 is associated with the user's recovered address.
</li>
<li>
If step 4 passes, the server provides the user with the requested content.
</li>
</ol>

##### Closing Remarks
This setup has one main advantage over existing patterns, in that requests for content for the same user can be differentiated. The new nonce value generated each time a content request is received prevents the user from consuming the content again. Content can only be re-consumed if the user updates the smart contract with a new payment and nonce. 

This pattern also doesn't care what address the user is claiming to be on the client side (with MetaMask for example). In fact there is no way for the client to tell the server who they are except through signing data. Server's should not be told who they are talking to but they should infer it themselves through a cryptographic challenge like signing data. 

I hope this helps anyone looking to add some pay-per-view functionality to content on their web applications!
