---
layout: post
title:  "Hashing and Signing Nano Blocks with C#"
date:   2018-04-08 21:53:51 +0100
categories: nano
---

[Previously](/nano/2018/03/22/generating-nano-addresses.html) I looked at deriving private keys, public keys and addresses from Nano wallet seeds. This knowledge is of no use if we don't know how to create and sign a Nano block. There are currently five types of Nano block, the original four block types and the new universal block here we will look at just the original four block types, with a post on the uninversal blocks coming soon after. 

The four types of block we are going to look at are **Send**, **Receive**, **Change** and **Open**. First we will show how to create the hash of each block type and then how we go about signing the blocks. Signing the blocks proves to the network that the creator of the block is the account owner, or holder of the private key.

# Send blocks

A send block is the record of value being sent from the account to another account. To hash a send block we need three things:

* The previous block hash in the sending account's blockchain.
* The remaining account balance after sending the Nano.
* The address of the account the Nano is being sent to.

By looking at the first two items, the system can determine the anount being sent, this is because we can see the balance of the account at the previous block and subtract the remaining balance (item 2) from it. By storing the remaining balance in each send block, we need only to look at the latest send block and the value of all receive blocks after it to determine the account balance.

Nano uses Blake2b to calculate the hash of the blocks. In the case of the send block, we calculate the hash as follows:

`blake2b(previousHash + addressPublicKey + remainingBalance)` 

Lets see how we can do this in c#, first lets get the byte array representation of the three items to sign.

{% highlight c# %}

var previousHash = "1E1EB1C48A42FE1EE245342D8E66FEE5761AEB840FFFE9ACC10199AF6F73939E";
var address = "xrb_13cwinwfd8uq65nj5m3hhrt5tmcjmk4a3zu7d737311er1eg6jtsqxwdp4oc";
var balanceHex = "0000024840846ED118ABCC068DFFFFFF";

var previousBytes = HexStringToByteArray(previousHash);
var publicKey = AddressToPublicKey(address);
var balanceBytes = HexStringToByteArray(balanceHex.PadLeft(32,'0'));

{% endhighlight %}

Here I've used my helper methods in my NanoDotNet library [here](https://github.com/Flufd/NanoDotNet/blob/master/NanoDotNet/Utils/Utils.cs) to get the public key from a Nano formatted address and turn the hex strings into byte arrays. Note that we are left padding the balance with zeros to bring it up to 32 hexadecimal characters or 16 bytes/128 bits.

Next we need to pass our byte arrays through our blake2b hasher.
{% highlight c# %}

var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 32 });
blake.Init();
blake.Update(previousBytes);
blake.Update(publicKey);
blake.Update(balanceBytes);
var hashBytes = blake.Finish();

var hash =  ByteArrayToHex(hashBytes);
{% endhighlight %}

This gives us the hash for the send block. Lets take a look at the remaining block types that use a similar method to compute the hash.

# Receive blocks

A receive block is the record of receipt of a Nano amount into the account's block chain. By having a receive block we can define the order that the send blocks are received or pocketed into the account. This allows an account to even ignore a send block if it does not want to receive it, for example, to ignore small send amounts or sends from specific accounts. To sign a receive block we need to know just two things:

* The previous block hash in the receiving account's blockchain.
* The hash of the send block that we are receiving.

The call to the hash function for a receive block looks like this:

`blake2b(previousHash + sourceHash)`

Here is how it can be done in c#

{% highlight c# %}

var previousHash = "6D498FCB913BE2B3609C6F60B3A8939DFD254548E712E1BA667AE48AE4D06FCD";
var sourceHash = "BD7DEABCB119EF6C5DA0252DB233D4F8ACC7D2C270331D6108377FE5E9A09A7D";

var previousBytes = HexStringToByteArray(previousHash);
var sourceHashBytes = HexStringToByteArray(sourceHash);

var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 32 });
blake.Init();
blake.Update(previousBytes);
blake.Update(sourceHashBytes);
var hashBytes = blake.Finish();

var hash = ByteArrayToHex(hashBytes);
{% endhighlight %}

You can see that the process is very similar but we are just using different input fields for the different types of block.

# Change block

A change block allows an account to change it's representative. The representative will vote on behalf of the account when a fork occurs in the network. For a change block we need just two things to compute the hash:

* The previous block hash of the account.
* The account of the representative.

Again the values passed to the hashing function are very similar

`blake2b(previousHash + representativePublicKey)`

And in c#

{% highlight c# %}

var previousHash = "AE359BF914A620E9B07EFB8BD540B64A03972D102273AD4B6B0832BDC93B9D8E";
var representativeAccount = "xrb_3iuommeb5gpnhxbsubeswmbg5rdydfstzpih9qgmmfcnqmxs1bzeootdqryq";

var previousBytes = HexStringToByteArray(previousHash);
var representativePublicKey = AddressToPublicKey(representativeAccount);

var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 32 });
blake.Init();
blake.Update(previousBytes);
blake.Update(representativePublicKey);
var hashBytes = blake.Finish();

var hash = ByteArrayToHex(hashBytes);

{% endhighlight %}

# Open blocks

An open block is the first block in an account's blockchain. It is similar to both a receive and change block, and is kind of a combination of them. To open a Nano account we need to receive some Nano from a send block and set the account's representative at the same time. Unlike the other blocks we won't have a previous block for this one, as it will be the first one in the account! Here is what is required:

* Public key of the account.
* Hash of the send block that we are receiving the funds for.
* Public key of the account representative.

Here is the order we pass the values into the hash function:

`blake2b(sourceHash + representativePublicKey + accountPublicKey)`

Now lets see that in c#

{% highlight c# %}

var sourceHash = "B2EC73C1F503F47E051AD72ECB512C63BA8E1A0ACC2CEE4EA9A22FE1CBDB693F";
var representativeAccount = "xrb_1anrzcuwe64rwxzcco8dkhpyxpi8kd7zsjc1oeimpc3ppca4mrjtwnqposrs";
var accountAddress = "xrb_3arg3asgtigae3xckabaaewkx3bzsh7nwz7jkmjos79ihyaxwphhm6qgjps4";

var sourceBytes = HexStringToByteArray(sourceHash);
var representativePublicKey = AddressToPublicKey(representativeAccount);
var accountPublicKey = AddressToPublicKey(accountAddress);

var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 32 });
blake.Init();
blake.Update(sourceBytes);
blake.Update(representativePublicKey);
blake.Update(accountPublicKey);
var hashBytes = blake.Finish();

var hash = ByteArrayToHex(hashBytes);

{% endhighlight %}

Okay, now we know how to create the hashes for each block type. 

# Signing the blocks

For each block type we simply need to sign the hash of the block for it to be accepted by the network. Using the public and private key of the account, we use Ed25519 with blake2b as the hashing function to create the signature. 

Use the stored public and private key, or derive the public key from the private key

{% highlight c# %}

byte[] privateKey = ....;
var publicKey = Ed25519.PublicKey(privateKey);

{% endhighlight %}

Now sign the block

{% highlight c# %}

var signature = Ed25519.Signature(HexStringToByteArray(hash), privateKey, publicKey);
var signatureHex =  ByteArrayToHex(signature);

{% endhighlight %}

Again, I've added example methods to hash and sign each block type to the helper class in NanoDotNet [here](https://github.com/Flufd/NanoDotNet/blob/master/NanoDotNet/Utils/Utils.cs)

Next we will look at computing the proof of work for each block type.