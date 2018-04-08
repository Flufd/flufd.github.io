---
layout: post
title:  "Generating Nano addresses from seeds with C#"
date:   2018-03-22 22:07:51 +0000
categories: nano
---
# What is a Nano seed?

A Nano seed is a 64 character random hexadecimal string (32 bytes or 256 bits) that is used to generate the private keys for a Nano account. It is possibe to use another mechanism to generate the private key for a Nano account, but the official wallets use a seed. 

_Example Seed_

`9F0E444C69F77A49BD0BE89DB92C38FE713E0963165CCA12FAF5712D7657120F`

# What is a Nano address?

A nano address is a 64 character string starting with `xrb_` and ending with an 8 character checksum.

_Example Address_

`xrb_3yiqftdf6t4s9nwhtpjpqr1sd5yinyupa3m54fh7c1mxy53bkpecczshr4uy`

So how do we get from a seed to an address? There are three steps in the process:
- Combine the **seed** with an **index** to create a **private key** for the address
- Generate the **public key** for the address using the **private key**
- Encode the **public key** into the **Nano address format**

`seed + index -> private key -> public key -> nano address`

You can see how changing the seed affects the generation of the address on this [nanoo.tools page](https://nanoo.tools/key-address-seed-converter?type=seed&value=9F0E444C69F77A49BD0BE89DB92C38FE713E0963165CCA12FAF5712D7657120F)

# Generating the private key

Nano uses the Blake2b hashing algorthm to generate the private key from the seed. I'm using the c# implementation found here [https://github.com/BLAKE2/BLAKE2](https://github.com/BLAKE2/BLAKE2).

First lets get the bytes from the hex string. I'm using a helper method found on Stack Overflow [here](https://stackoverflow.com/a/321404), other methods are available.

{% highlight c# %}
public static byte[] HexStringToByteArray(string hex) {
    return Enumerable.Range(0, hex.Length)
                     .Where(x => x % 2 == 0)
                     .Select(x => Convert.ToByte(hex.Substring(x, 2), 16))
                     .ToArray();
}
{% endhighlight %}

{% highlight c# %}

var seed = "9F0E444C69F77A49BD0BE89DB92C38FE713E0963165CCA12FAF5712D7657120F";
uint index = 0;

var seedBytes = HexStringToByteArray(seed);

{% endhighlight %}

Put the seed through Blake2b with the index of the account you wish to generate. We need to check the endianess of the index bytes as the blake implementation is expecting the bytes in big endian in order to match the output of the official wallets.

{% highlight c# %}
var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 32 });
blake.Init();
blake.Update(seedBytes);
blake.Update(BitConverter.IsLittleEndian ? BitConverter.GetBytes(index).Reverse().ToArray() : BitConverter.GetBytes(index));
var privateKey = blake.Finish();
{% endhighlight %}

Great! Now we have the private key for the account. Using a different index value here would produce a different output. This means that a single seed is able to generate many accounts, and this process is deterministic, which is why you are able to enter the same seed into two different implementations of a wallet and get access to the same account. Of course, hashing the seed and index is just one way to create private keys, what I'm showing you here is how to get the same output as the official wallets, but you could generate the private key in your own way if you wanted to.

# Generating the public key

The public key is derived from the private key of the account. Nano uses the [Ed25519](https://ed25519.cr.yp.to/) signature scheme with the SHA512 hash replaced with Blake2b. I am using [this](https://github.com/hanswolff/ed25519) library to do my signing, altered by changing the `ComputeHash` method to use the Blake2b implementation instead.

*Ed25519.cs*
{% highlight c# %}
private static byte[] ComputeHash(byte[] m)
{
    var hasher = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig { OutputSizeInBytes = 64 });
    hasher.Init();
    hasher.Update(m);
    return hasher.Finish();                
}
{% endhighlight %}

Pass the private key to the Ed25519 implementation to get the public key.

{% highlight c# %}
var publicKey = Ed25519.PublicKey(privateKey);
{% endhighlight %}

Now we have both the public and private key, we could use these to sign a Nano block.

# Encoding the public key in the Nano address format

The Nano address is formatted so that it only contains characters that are easily distniguishable from one another, for example the letter `l` is removed as well as the number `0`. Each character encodes 5 bytes of the public key, apart from the first character as the public key is padded with zeros to bring the public key bit length up to a multiple of 5.

![Nano Address Encoding]({{ "/assets/images/nano_address_encoding.png" | absolute_url }})

This image from the [Exchange integration Guide](https://cdn.discordapp.com/attachments/370285507185344524/375275437527662593/RaiBlocks_Exchange_Integration_Guide_rev1.pdf) posted on the Nano discord shows the exact encoding for each character. Note that there is a slight error on this image; 00110 actually encodes to 8 and 00111 encodes to 9, instead of "a" and "b".

The last 8 characters act as a checksum for the address. This checksum is computed as a 5 byte hash of the public key, reversed.

Lets compute the checksum of the address
{% highlight c# %}
var blake = Blake2Sharp.Blake2B.Create(new Blake2Sharp.Blake2BConfig() { OutputSizeInBytes = 5 });
blake.Init();
blake.Update(publicKey);
var checksumBytes = blake.Finish().Reverse().ToArray();
{% endhighlight %}

Now, to encode the address we will need a mapping of 5 bit numbers to characters. I am going to convert the 5 bit numbers into strings and use those as my map key.
{% highlight c# %}
xrb_addressDecoding = new Dictionary<string, char>();
var i = 0;
foreach (var validAddressChar in "13456789abcdefghijkmnopqrstuwxyz")
{
    xrb_addressDecoding[Convert.ToString(i, 2).PadLeft(5, '0')] = validAddressChar;
    i++;
}
{% endhighlight %}

Write a function to encode the bytes as we will be doing this twice. Here I've added an optional parameter to determine if the binary string is zero padded before encoding. The function creates a string of the bits e.g. 

`00001001|00101001|01001010|101....` 

from the passed bytes and then interprets the result 5 bits at a time to convert to the correct character, e.g. 

`00001|00100|10100|10100|10101|01....`.

{% highlight c# %}
 private static string XrbEncode(byte[] bytes, bool padZeros = true)
 {
    var binaryString = padZeros ? "0000" : "";
    for (int i = 0; i < bytes.Length; i++)
    {
        binaryString += Convert.ToString(bytes[i], 2).PadLeft(8, '0');
    }

    var result = "";

    for (int i = 0; i < binaryString.Length; i += 5)
    {
        result += xrb_addressDecoding[binaryString.Substring(i, 5)];
    }

    return result;
}
{% endhighlight %}

Now we can concatenate all of the address parts together
{% highlight c# %}
var address = "xrb_" + XrbEncode(publicKey) + XrbEncode(checksumBytes, false); 
{% endhighlight %}

And that's it! You can see how I've used these techniques to implement my helper classes in the Utils.cs class in my Nano library [here](https://github.com/Flufd/NanoDotNet/blob/master/NanoDotNet/Utils/Utils.cs)
