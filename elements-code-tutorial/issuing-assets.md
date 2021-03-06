---
layout: page
title: Issuing assets
permalink: /elements-code-tutorial/issuing-assets
---

# Elements code tutorial

## Issuing your own assets

Previously, the RPC calls we've made always returned values denominated in "bitcoin". This is simply because that is what the labelling of the initial asset created by Elements defaults to. 

##### NOTE: You can use the **defaultpeggedassetname** startup paramater to change the label of the default asset to something other than "bitcoin". You can also change the initial issuance amount of the default asset using the **initialfreecoins** parameter and the **initialreissuancetokens** parameter to allocate reissuance tokens for the default asset.

In this section we'll issue our own assets, label them, re-issue them (basically create some more of them) and learn how to send them to other addresses. We'll also take a look at how to keep track of what assets have been issued and re-issued and also how to destroy assets. This last feature may be something which is required if you implement your own blockchain solution based upon real world assets. More on that later.

First, let's take a look at Alice's wallet to remind ourselves what it currently holds. The "getwalletinfo" command accepts an optional asset type parameter and running it without this returns all known assets, as we have been doing previously:

~~~~
e1-cli getwalletinfo
~~~~

We see that Alice holds a lot of the "bitcoin" asset and nothing else:

<div class="console-output">"balance": {
    "bitcoin": 10500001.00000000
</div>

Running the command again with a parameter of "bitcoin" should therefore show the same thing:

~~~~
e1-cli getwalletinfo bitcoin
~~~~

Which it does, albeit in a slightly different format.

Every asset you issue within Elements (including the "bitcoin" default) will be assigned its own hex value. This is used to uniquely identify it on the network. Notice how "bitcoin" is displayed with a readable asset name however. This is because Elements automatically associates the label "bitcoin" with the asset hex for that default asset. To find out its hex value we can run: 

~~~~
e1-cli dumpassetlabels
~~~~

Which returns:

<div class="console-output">"bitcoin": "b2e15d0d7a0c94e4e2ce0fe6e8691b9e451377f6e46e8045a86f7c4b5d4f0f23"
</div>

We can also use the asset's hex value as a command parameter instead of its label:

~~~~
e1-cli getwalletinfo b2e15d0d7a0c94e4e2ce0fe6e8691b9e451377f6e46e8045a86f7c4b5d4f0f23
~~~~

One of the main features of Elements is the ability to issue your own assets. We'll do this next and then look at the details using some of the commands we've already used. 

#### Note: There is nothing inherently different between assets in the way they are handled within the Elements protocol.

Run the following to issue a quantity of 100 of a new asset.

~~~~
ISSUE=$(e1-cli issueasset 100 1)
~~~~

That will create a new asset type, an initial supply of 100 and also 1 **reissuance token**. The reissuance token is used to prove authority to reissue more of the asset at a later date. We have issued one such token in the command above. The token is transferable and you can initially create as many as you think you will need based upon how many of the network participants will need to perform this duty. The token is used to provide proof that any transactions that create new amounts of the asset were sent by someone holding the required authority. Each asset has its own reissuance token. We'll look at this in more detail later.

##### NOTE: When you create the reissuance token as above, where the amount was "1", you are actually creating 100,000,000 of them when measured in their smallest denomination. This is because they are divisible like every other asset on Elements. The "1" is little more than a user interface concession. For ease of readability we will refer to this issuance as "one token".

We have stored all of the returned data from the issuance command in a variable named "ISSUE", which we'll pull the hex of the new asset from, storing that value in another variable named "ASSET". We'll also store the "token" value (which we'll explain and use later) and the "txid" and "vin" of the issuance, which will be used when we try and unblind the issuing transaction shortly.

In order to do this we can use a tool called [jq](https://stedolan.github.io/jq/) (which we installed as part of the tutorial dependencies) to strip out and store only the parts returned and saved within "ISSUE" that we are interested in:

~~~~
ASSET=$(echo $ISSUE | jq '.asset' | tr -d '"')
TOKEN=$(echo $ISSUE | jq '.token' | tr -d '"')
ITXID=$(echo $ISSUE | jq '.txid' | tr -d '"')
IVIN=$(echo $ISSUE | jq '.vin' | tr -d '"')
~~~~

To see the hex identifier for the asset we issued run:

~~~~
echo $ASSET
~~~~

Which will return something like this:

<div class="console-output">f0379482f9b77917670be0f060cc58debc6d93db0bf857458d5fb19080c8ab67
</div> 

In order to view all asset issuances that have been made we run the 'listissuances" command:

~~~~
e1-cli listissuances
~~~~

That will show two instances of issuances. One will be the original default issuance of an asset with the "assetlabel" of "bitcoin" and the one that we have just issued ourselves. You'll notice that both have the following:

<div class="console-output">"isreissuance": false,
</div>

This indicates that both entries in the list are original issuances and not reissuances. More on this soon. You'll also see that the newly issued asset does not have an "assetlabel". 

#### Note: Asset labels are **not** part of network protocol consensus and are local only to each node. You should not rely on them for transaction processing but instead use the asset's hex value, which is shared across the network.

You can set the label by assigning it against the hex identifier of the asset. This can be done in the relevant elements.conf file by adding a line:

<div class="console-output">assetdir=asset_hex_here:your_label_here
</div>

Or you can do this by passing in "assetdir" as a parameter when you start the node. We'll do this now and call our new asset "demoasset":

~~~~
e1-cli stop
e1-dae -assetdir=$ASSET:demoasset
e1-cli listissuances
~~~~

This shows that the asset we issued has the label we assigned to its hex value:

<div class="console-output">"assetlabel": "demoasset",
</div>

Having labelled our asset for ease of reference, we will now look at the issuance data for "demoasset" in more detail. You will notice a "token" property similar to that below:

<div class="console-output">"token": "33244cc19dd9df0fd901e27246e3413c8f6a560451e2f3721fb6f636791087c7",
</div>

This is the hex of the token and it can be used to reissue the asset. Yours will likely differ from the actual value above. There is also a "tokenamount" property which corresponds to the amount we created:

<div class="console-output">"tokenamount": 1.00000000,
</div>

Notice that the default "bitcoin" asset has a token hex but that the token amount is 0, meaning that it cannot be reissued. This can be changed by setting the **initialreissuancetokens** parameter to a non-zero amount when you first initialize a chain.

Confirm the transaction:

~~~~
e1-cli generate 1
~~~~

Then wait a few seconds before having Bob's wallet list its view of the asset issuances:

~~~~
e2-cli listissuances
~~~~

Bob's wallet isn't aware of the issuance, so we'll import the address into his wallet:

~~~~
IADDR=$(e1-cli gettransaction $ITXID | jq '.details[0].address' | tr -d '"')
e2-cli importaddress $IADDR
~~~~

If we try and view the list of issuances from Bob's node now we'll see the issuance, but notice that the amount of the asset and the amount of its associated token are hidden:

~~~~
e2-cli listissuances
~~~~

The asset amount and the token amount are both blinded and show as -1:

<div class="console-output">"tokenamount": -1,
"assetamount": -1,
</div>

Earlier in the tutorial we were able to expose the amount and type of asset being sent in a regular Confidential Transaction by exporting the blinding key used to create the blinded address and importing it into another wallet. We can do the same type of thing with the issuance transaction using the **issuance blinding key**.

First, we need to export the issuance blinding key. We refer to issuances by their txid/vin pair. As there is only one per input it will be zero, but we'll use the value we saved earlier as it is good practice to not rely on such things staying fixed:

~~~~
ISSUEKEY=$(e1-cli dumpissuanceblindingkey $ITXID $IVIN)
e2-cli importissuanceblindingkey $ITXID $IVIN $ISSUEKEY
~~~~

Now when we run the command to list known issuances from Bob's wallet we should see the actual values:

~~~~
e2-cli listissuances
~~~~

Which returns: 

<div class="console-output">"tokenamount": 1.00000000,
"assetamount": 100.00000000,
</div>

Indeed, Bob's wallet can now see both the amount of the asset and its reissuance token that were issued.

Just like any other asset in Elements, we can send our "demoasset" from Alice's address to Bob's using the "sendtoaddress" command. This differs from its implementation in Bitcoin's source code in that it accepts an additional parameter, which allows you to specify the type of asset to be sent. Be aware that the step above where we imported the issuance blinding key is not required in order to transact an issued asset between addresses and wallets. Importing the issuance blinding key just enables another wallet to view the issuance details in full.

~~~~
E2DEMOADD=$(e2-cli getnewaddress)
e1-cli sendtoaddress $E2DEMOADD 10 "" "" false demoasset
e1-cli generate 1
~~~~

##### NOTE: The parameters you can pass to "sendtoaddress" are detailed within the elements/src/wallet/rpcwallet.cpp file on [Github](https://github.com/ElementsProject/elements)

Bob's wallet now has an amount of 10 "demoasset" and Alice has 90:

~~~~
e2-cli getwalletinfo
e1-cli getwalletinfo
~~~~

As we didn't assign a label in Bob's node for the asset we created, it will be identified by its hex value instead. We will therefore have to use the hex identifier instead of the asset label when we send it from his node. Remember that asset labels are local only to each node and are not part of the network's protocol rules. We'll demonstrate how Bob can send the asset using the hex value by transferring the 10 "demoasset" back to Alice:

~~~~
E1DEMOADD=$(e1-cli getnewaddress)
e2-cli sendtoaddress $E1DEMOADD 10 "" "" false $ASSET
e2-cli generate 1
~~~~

We should see that Bob's wallet has no "demoasset" in it anymore and Alice's is back to 100:

~~~~
e2-cli getwalletinfo
e1-cli getwalletinfo
~~~~

We can see that is indeed the case. 

In the next section we will look in detail at how to reissue more of an asset.

[Next: Reissuing assets]({{ site.url }}/elements-code-tutorial/reissuing-assets)

