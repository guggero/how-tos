# HOW TO create a secure multi crypto currency wallet and back it up safely

In this HOW TO I will show you how you can set up a Wallet that will hold multiple Crypto Currencies. 
I will then propose a way how to securely back that wallet up and store the backups for long-term keeping.

## Tools we will be using

Most of the tools we are going to use have been written by me. But you don't have to trust me! 
They are all open source and [hosted on GitHub](https://github.com/guggero). 
You can download them and use them offline (for example the Blockchain Demo).
Or you can run them on your own server (for example the Cloudcoins service). 

* [Hierarchical Deterministic Wallet tool](https://gugger.guru/blockchain-demo/#!/hd-wallet):
  Part of my Blockchain Demo, can be used offline, [code is here](https://github.com/guggero/blockchain-demo).  
  We'll be using this to generate our keys.
* [www.cloudcoins.ch](https://www.cloudcoins.ch): My public zero-knowledge key storage service.
  Can be run in a Docker container, [code is here](https://github.com/guggero/cloudcoins).  
  We'll be using this to store one of the backups.
* [(optional) Coinomi](https://coinomi.com): Multi-Asset Wallet App, developed by Coinomi Ltd. 
  Available for Android, iOS coming soon.

# Create your Root Key

In this step we are going to create a so called [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
master extended key, also called root key or HD root node key. 
From this key, all your addresses are going to be derived.
Therefore you only need to backup and protect this key, not every single address you will use to move your coins.

But obviously, if someone is able to get this key from you, they will bei able to control all your funds and steal
all your coins. That's why we are not just going to create some random words but also encrypt those with a password.
That way you can control how you store/backup these separate pieces of information.

But let's start now:
1. Go to the [HD Wallet tool](https://gugger.guru/blockchain-demo/#!/hd-wallet), or, if you're not sure about the
  security of your computer (or my my tool), download the code and run it offline by opening the index.html file. 
  There is a warning about using a browser to generate random numbers. If you are using a recent version of Chrome or
  Firefox, you don't need to worry about that since they have implemented a strong pseudo-random number generator. 
  Also, we are going to encrypt the generated phrase with a password. 
  If you are using Internet Explorer or Edge, you sould not be buying Crypto Currencies at all and just give away your
  money to charity!
1. You see 12 random english words already pre-filled in the "Seed Mnemonic" field. Click on "Generate new" as many
  times as you like to generate new random words. **DO NOT use your own words**, if they aren't in the BIP39 dictionary,
  you might end up with worse entropy than the pseudo-random generator of your browser. 
1. Once you're satisfied with some 12 words, **WRITE THEM DOWN**. Yes, on paper!
1. Now, think of a password/passphrase that you will most likely still remember in years. This phrase will protect your
  12 words. For example, if someone finds the paper you have just written down the words on, they could spend all
  your coins. With the password you prevent that from happening. But of course, if you write down the password
  together with your 12 words, you don't add any extra security. So what I suggest is: You write down a *hint* to
  your password together with the 12 words. So in 10 years when you find your paper backup, you still have a clue
  to what password you used. If you forget your password, nobody can reset or recover it for you!
1. Make sure that the selected "Method" is set to "BIP39 default (like Coinomi)"
1. Now you should see a long string in the field "HD root node key base58", starting with `xprv`. This is your root key.
  It is calculated from the 12 words and your password. That's why you should never write that down or copy to any
  medium/app you don't completely trust! All your coins will be spendable from that key!

# Make a backup in the cloudcoins zero-knowledge storage service

One copy of anything is never enough to really call it a backup. So go ahead and make another copy of the paper you just
wrote everything down on and put it in a (fire proof) safe.

Now you have a backup! But one more on another medium is always a good idea. So here is another way to securely store
your key while still being able to generate addresses from it (and export private keys of these addresses if needed).

1. Now go to [www.cloudcoins.ch](https://www.cloudcoins.ch) and create an account. You can use the same password you
  used in the previous step to setup the account. But of course, it's always a risk to use a password multiple times.
  But here it is much more important that you will be able to remember the password in the future as there is nobody
  that can reset or recover it for you if you forget it!
1. After creating the account, log in and go to "My keychains". Create a keychain, give it any name and select the tab
  "Import coinomi recovery phrase". Now enter the 12 words and your password. The "Derived BIP32 root key" should be
  exactly the same as seen in the other application before.
1. By clicking "Create keychain" you store the root key in the database of cloudcoins.ch. But the key is only stored
  heavily encrypted (AES256) with your password you used to create the account. The unencrypted key never leaves your
  computer! If you want to enter paranoid mode, you can run your own version of cloudcoins on an offline computer.
1. Now that you have a keychain, you can generate addresses for your coins. For example you can click on
  "Add coin/crypto currency" and select "Bitcoin (BTC)". The tool will show you an address that you can send your
  Bitcoin to that is in your control, e.g. where you have the private key for. The service cloudcoins is mainly meant
  as another piece of backup but more accessible than real "cold storage".
  Cloudcoins is still very much in development so the usability will improve over time. Contact me if you have any
  suggestions!

# (optional) Import key into Coinomi

If you want to be able to spend your coins and see real-time balance, you can also import the key we have created into
the Coinomi App. Follow the instructions in the app to import/restore your wallet.

The App will also ask you for 12 to 15 words (enter those written down in step 1) and a passphrase.