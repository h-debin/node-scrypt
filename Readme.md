#Scrypt For NodeJS

[![Build Status](https://travis-ci.org/barrysteyn/node-scrypt.png?branch=master)](https://travis-ci.org/barrysteyn/node-scrypt)

node-scrypt is a native node C++ wrapper for Colin Percival's Scrypt utility. 

##Table Of Contents
 * [Scrypt](#scrypt)
 * [Introducing Node-scrypt version 2.X](#introducing-node-scrypt-version-2)
 * [API](#api)
 * [Example Usage](#example-usage)
 * [FAQ](#faq)
 * [Credits](#credits)

##Scrypt
Scrypt is an advanced crypto library used mainly for [key derivation](http://en.wikipedia.org/wiki/Key_derivation_function): More information can be found here:

* [Tarsnap blurb about Scrypt](http://www.tarsnap.com/scrypt.html) - Colin Percival (the author of Scrypt) explains a bit about it.
* [Academic paper explaining Scrypt](http://www.tarsnap.com/scrypt/scrypt.pdf).
* [Wikipedia Article on Scrypt](http://en.wikipedia.org/wiki/Scrypt).

##Introducing Node-Scrypt Version 2
This module is a complete rewrite of the previous module. It's main highlights are:
 * Access to the underlying key derivation function
 * Extensive use of node's buffers
 * Easy configuration
 * Removal of scrypt encryption/decryption (this will soon be moved to another module)

The module consists of four functions:
 1. params - a translation function that produces scrypt parameters
 2. hash - produces a 256 bit hash using scrypt's key derivation function
 3. verify - verify's a hash produced by this module
 4. kdf - scrypt's underlying key dervivation function

Each function has a member json object called *config* used to configure settings. All the functions (except the params function) can accept th following encodings:
 1. ascii
 2. utf8
 3. base64
 4. ucs2 
 5. binary
 6. hex
 7. buffer

The last encoding is node's [Buffer](http://nodejs.org/api/buffer.html) object. Buffer is useful for representing raw binary data and has the ability to translate into any of the encodings mentioned above. It is for these reasons that encodings default to buffer in this module.

##Params
This function translates human understandable parameters to Scrypt's internal parameters. 

The human understandable parameters are as follows:
 1. **maxtime**: the maximum amount of time scrypt will spend when computing the derived key.
 2. **maxmemfrac**: the maximum fraction of the available RAM used when computing the derived key.
 3. **maxmem**: the maximum number of bytes of RAM used when computing the derived encryption key. 

Scrypt's internal parameters are as follows:
 1. **N** - general work factor, iteration count.
 2. **r** - blocksize in use for underlying hash; fine-tunes the relative memory-cost.
 3. **p** - parallelization factor; fine-tunes the relative cpu-cost.

###A Note On How Memory Is Calculated
`maxmem` is often defaulted to `0`. This does not mean that `0` RAM is used. Instead, memory used is calculated like so (quote from Colin Percival):

> the system [will use] the amount of RAM which [is] specified [as the] fraction of the available RAM, but no more than maxmem, and no less than 1MiB

Therefore at the very least, 1MiB of ram will be used.

##Hash
The hash function does the following:
 * Adds random salt.
 * Creates a HMAC to protect against active attack.
 * Uses the Scrypt key derivation function to derive a hash for a key.

###Hash Format
All hashes start with the word *"scrypt"*. Next comes the scrypt parameters used in the key derivation function, followed by random salt. Finally, a 256 bit HMAC of previous content is appended, with the key for the HMAC being produced by the scrypt key derivation function. The result is a 768 bit (96 byte) output:
 1. bytes 0-5: The word *"scrypt"*
 2. bytes 6-15: Scrypt parameters N, r, and p
 3. bytes 16-47: 32 bits of random salt
 4. bytes 48-63: A 16 bit checksum
 5. bytes 64-95: A 32 bit HMAC of bytes 0 to 63 using a key produced by the Scrypt key derivation function.

Bytes 0 to 63 are left in plaintext. This is necessary as these bytes contain metadata needed for verifying the hash. This information not being encrypted does not mean that security is weakened. What is essential in terms of security is hash **integrity** (meaning that no part of the hashed output can be changed) and that the original password cannot be determined from the hashed output (this is why you are using Scrypt - because it does this in a good way). Bytes 64 to 95 is where all this happens.

##Verify
The verify function takes two inputs:
 1. **hash** a hash produced by the hash function
 2. **key**  a key. 

It will verify whether the hash can be derived from the key and return a boolean result.

##Key Derivation Function
The underlying Scrypt key derivation function. This functionality is exposed for users who are quite experienced and need the function for business logic. A good example is [litecoin](https://litecoin.org/) which uses the scrypt key derivation function as a proof of work. The key derivation function in this module is tested against [three of the four test vectors](http://tools.ietf.org/html/draft-josefsson-scrypt-kdf-00#page-11) in the original scrypt paper. The fourth test vector takes too long to computer and is infeasible to use as testing for continuous integration. Nevertheless, it is included in the tests, but commented out - uncomment it and run the tests, but be warned that it is rather taxing on resources.

###Use Hash To Store Keys
If your interested in this module is to produce hashes to store passwords, then I strongly encourage you to use the hash function. The key derivation function does not produce any [message authentication code](http://en.wikipedia.org/wiki/Message_authentication_code) to ensure integrity. You will also have to store the scrypt parameters separately. 

In short: If you are going to use this module to store keys, then use the hash function. It has been customised for general key storage and is both easier to use and provides better protection compared to the key derivation function.

##Backward Compatibility For User's Of Version 1.x
Four extra functions are provided for means of backward compatibility:
 1. passwordHash
 2. passwordHashSync
 3. verifyHash
 4. verifyHashSync

The above functions are defaulted to behave exactly like the previous version.

#API
##Hash
###Config Object

#Example Usage

# FAQ
## General
### What Platforms Are Supported?
This module supports most posix platforms. It has been tested on the following platforms: **Linux**, **MAC OS** and **SmartOS** (so its ready for Joyent Cloud). This includes FreeBSD, OpenBSD, SunOS etc.
### What About Windows?
Windows support is not native to Scrypt, but it does work when using cygwin. With this in mind, I will be updating this module to work on Windows with a prerequisite of cygwin. 
## Scrypt
### What Is Scrypt?
### What Are The Three Scrypt Parameters (*N*, *r* and *p*)
## Hash
### What Are The Essential Properties For Storing Passwords
Storing passwords requires three essential properties

* The password must not be stored in plaintext. (Therefore it is hashed).
* The password hash must be salted. (Making a rainbow table attack very difficult to pull off).
* The salted hash function must not be fast. (If someone does get hold of the salted hashes, their only option will be brute force which will be very slow).

As an example of how storing passwords can be done badly, take [LinkedIn](http://www.linkedin.com). In 2012, they [came under fire](http://thenextweb.com/socialmedia/2012/06/06/bad-day-for-linkedin-6-5-million-hashed-passwords-reportedly-leaked-change-yours-now/#!rS1HT) for using unsalted hashes to store their passwords. As most commentators at the time were focusing no salt being present, the big picture was missed. In fact, their biggest problem was that they used [sha1](http://en.wikipedia.org/wiki/SHA-1), a very fast hash function.

### If random salts are used for each hash, why does each hash start with *c2NyeXB0* when using passwordHash
All hashes start with the word *"scrypt"*. The reason for this is because I am sticking to Colin Percival's (the creator of Scrypt) hash reference implementation whereby he starts off each hash this way. The base64 encoding of the ascii *"scrypt"* is *c2NyeXB0*. Seeing as *passwordHash* defaults it's output to base64, every hash produced will start with *c2NyeXB0*. Next is the Scrypt parameter. Users of Scrypt normally do not change this information once it is settled upon (hence this will also look the same for each hash). 

To illustrate with an example, I have hashed two password: *password1* and *password2*. Their outputs are as follows:

    password1
    c2NyeXB0AAwAAAAIAAAAAcQ0zwp7QNLklxCn14vB75AYWDIrrT9I/7F9+lVGBfKN/1TH2hs
    /HboSy1ptzN0YzHJhC7PZIEPQzf2nuoaqVZg8VkKEJlo8/QaH7qjU2VwB
    
    password2
    c2NyeXB0AAwAAAAIAAAAAZ/+bp8gWcTZgEC7YQZeLLyxFeKRRdDkwbaGeFC0NkdUr/YFAWY
    /UwdOH4i/PxW48fXeXBDOTvGWtS3lLUgzNM0PlJbXhMOGd2bke0PvTSnW

As one can see from the above example, both hashes start off by looking similar (they both start with *c2NyeXB0AAwAAAAIAAAAA* - as explained above), but afterwards, things change very rapidly. In fact, I hashed the password *password1* again:

    password1
    c2NyeXB0AAwAAAAIAAAAATpP+fdQAryDiRmCmcoOrZa2mZ049KdbA/ofTTrATQQ+m
    0L/gR811d0WQyip6p2skXVEMz2+8U+xGryFu2p0yzfCxYLUrAaIzaZELkN2M6k0

Compare this hash to the one above. Even though they start off looking similar, their outputs are vastly different (even though it is the same password being hashed). This is because of the **random** salt that has been added, ensuring that no two hashes will ever be indentical, even if the password that is being hashed is the same.

For those that are curious or paranoid, please look at how the hash is both [produced](https://github.com/barrysteyn/node-scrypt/blob/master/src/passwordhash/scrypthash.c#L146-197) and [verified](https://github.com/barrysteyn/node-scrypt/blob/master/src/passwordhash/scrypthash.c#L199-238) (you are going to need some knowledge of the [C language](http://c.learncodethehardway.org/book/) for this). 

##What Is Scrypt? 

###The Three Essential Properties Of Password Key Derivation
This Scrypt library automatically handles the above properties. The last item seems strange: Computer scientists are normally pre-occupied with making things fast. Yet it is this property that sets Scrypt apart from the competition. As computers evolve and get more powerful, they are able to attack this property more efficiently. This has become especially apparent with the rise of parallel programming. Scrypt aims to defend against all types of attacks, not matter the attackers power now or in the future.

### What This Module Provides
This module implements the following:

 * **Scrypt password key derivation**
    * All three essential properties of password key derivation are implemented (as described above).
    * Both *asynchronous* and *synchronous* versions are available.
 * **Scrypt encryption**
    * Both *asynchronous* and *synchronous* versions are available.

I suspect Scrypt will be used mainly as a password key derivation function (its author's intended use), but I have also ported the Scrypt encryption and decryption functions as implementations for them were available from the author. Performing Scrypt cryptography is done if you value security over speed. Scrypt is more secure than a vanilla block cipher (e.g. AES) but it is much slower. It is also the basis for the key derivation functions.

### The Scrypt Hash Format
##Why Use Scrypt?
It is probably the most advanced key derivation function available. This is is quote taken from a comment in hacker news:

>Passwords hashed with scrypt with sufficiently-high strength values (there are 3 tweakable input numbers) are fundamentally impervious to being cracked. I use the word "fundamental" in the literal sense, here; even if you had the resources of a large country, you would not be able to design any hardware (whether it be GPU hardware, custom-designed hardware, or otherwise) which could crack these hashes. Ever. (For sufficiently-small definitions of "ever". At the very least "within your lifetime"; probably far longer.)

The *three tweakable* inputs mentioned above are as follows (quoting from Scrypt's author Colin Percival):


Values for *maxtime*, *maxmemfrac* and *maxmem* are translated into the above values, which are then fed to the Scrypt function. The translation function also takes into account the CPU and Memory capabilities of a machine. Therefore values of *N*, *r* and *p* may differ for different machines that have different specs.

## Pros And Cons
Here are some pros and cons for using it:

###Pros

* The Scrypt algorithm has been published by [IETF](http://en.wikipedia.org/wiki/IETF) as an [Internet Draft](http://en.wikipedia.org/wiki/Internet_Draft) and is thus on track to becoming a standard. See [here](https://tools.ietf.org/html/draft-josefsson-scrypt-kdf-00) for the draft.
* It is being actively used in production at [Tarsnap](http://www.tarsnap.com/).
* It is much more secure than bcrypt.
* It is designed to be future proof against attacks with future (and more advanced) hardware.
* It is designed to defend against large scale custom hardware attacks.
* It is production ready.
* There is a Scrypt library for most major scripting languages (Python, Ruby etc). Now this module provides the library for NodeJS :)

I will end this section with a quote from Colin Percival (author of Scrypt):

> We estimate that on modern (2009) hardware, if 5 seconds are spent computing a derived key, the cost of a hardware brute-force attack against scrypt is roughly 4000 times greater than the cost of a similar attack against bcrypt (to find the same password), and 20000 times greater than a similar attack against PBKDF2.

###Cons
There is just one con I can think of: It is a relatively new library (only been around since 2009). Cryptographers don't really like new libraries for production deployment as it has not been *battle tested*. That being said, it is being actively used in [Tarsnap](http://www.tarsnap.com/) (as mentioned above) and the author is very active.

#Security Issues/Concerns
As should be the case with any security tool, this library should be scrutinized by anyone using it. If you find or suspect an issue with the code- please bring it to my attention and I'll spend some time trying to make sure that this tool is as secure as possible.

#Installation Instruction
##From NPM

    npm install scrypt

##From Source
You will need `node-gyp` to get this to work (install it if you don't have it: `npm install -g node-gyp`):

    git clone https://github.com/barrysteyn/node-scrypt.git
    cd node-scrypt
    node-gyp configure build

#Testing
To test, go to the folder where Scrypt was installed, and type:

    npm test

#Hash Info
All Scrypt output is encoded into Base64 using [René Nyffenegger](http://www.adp-gmbh.ch/) [library](http://www.adp-gmbh.ch/cpp/common/base64.html). The character sets that compromises all output are `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/`.

#Usage
There are both asynchronous and synchronous functions available. It is highly recommended not to use the synchronous version unless necessary due to the fact that Node's event loop will be blocked for the duration of these purposefully slow functions.

##Asynchronous Authentication
For interactive authentication, set `maxtime` to `0.1` - 100 milliseconds. 
   
###To create a password hash
 
    var scrypt = require("scrypt");
    var password = "This is a password";
    var maxtime = 0.1;

    scrypt.passwordHash(password, maxtime, function(err, pwdhash) {
        if (!err) {
            //pwdhash should now be stored in the database
        }
    });

Note: `maxmem` and `maxmemfrac` can also be passed to hash function. If they are not passed, then `maxmem` defaults to `0` and `maxmemfrac` defaults to `0.5`. If these values are to be passed, then they must be passed after `maxtime`  and before the callback function like so:
    
    var scrypt = require("scrypt");
    var password = "This is a password";
    var maxtime = 0.1;
    var maxmem = 0, maxmemfrac = 0.5;

    scrypt.passwordHash(password, maxtime, maxmem, maxmemfrac, function(err, pwdhash) {
        if (!err) {
            //pwdhash should now be stored in the database
        }
    });

###To verify a password hash

    var scrypt = require("scrypt");
    var password = "This is a password";
    var hash; //This should be obtained from the database

    scrypt.verifyHash(hash, password, function(err, result) {
        if (!err)
            return result; //Will be True
        
        return False;    
    });

##Synchronous Authentication
Again, for interactive authentication, set `maxtime` to `0.1` - 100 milliseconds. 
   
###To create a password hash
 
    var scrypt = require("scrypt");
    var password = "This is a password";
    var maxtime = 0.1;

    var hash = scrypt.passwordHashSync(password, maxtime);

Note: `maxmem` and `maxmemfrac` can also be passed to hash function. If they are not passed, then `maxmem` defaults to `0` and `maxmemfrac` defaults to `0.5`. If these values are to be passed, then they must be passed after `maxtime`  and before the callback function like so:
    
    var scrypt = require("scrypt");
    var password = "This is a password";
    var maxtime = 0.1;
    var maxmem = 0, maxmemfrac = 0.5;

    var hash = scrypt.passwordHashSync(password, maxtime, maxmem, maxmemfrac);

###To verify a password hash

    var scrypt = require("scrypt");
    var password = "This is a password";
    var hash; //This should be obtained from the database

    var result = scrypt.verifyHashSync(hash, password);

Note: There is no error description for the synchronous version. Therefore, if an error occurs, it will just return its result as `false`.

##Asynchronous Encryption and Decryption

    var scrypt = require("scrypt");
    var message = "Hello World";
    var password = "Pass";
    var maxtime = 1.0;

    scrypt.encrypt(message, password, maxtime, function(err, cipher) {
        console.log(cipher);
        scrypt.decrypt(cipher, password, maxtime, function(err, msg) {
            console.log(msg);
        });
    });

Note that `maxmem` and `maxmemfrac` can also be passed to the functions. If they are not passed, then `maxmem` defaults to `0` and `maxmemfrac` defaults to `0.5`. If these values are to be passed, then they must be passed after `maxtime`  and before the callback function like so:
    
    var scrypt = require("scrypt");
    var message = "Hello World";
    var password = "Pass";
    var maxtime = 1.0;
    var maxmem = 1; //Defaults to 0 if not set
    var maxmemfrac = 1.5; //Defaults to 0.5 if not set

    scrypt.encrypt(message, password, maxtime, maxmem, maxmemfrac, function(err, cipher) {
        console.log(cipher);
        scrypt.decrypt(cipher, password, maxtime, maxmem, maxmemfrac, function(err, msg) {
            console.log(msg);
        });
    });

##Synchronous Encryption and Decryption

    var scrypt = require("scrypt");
    var message = "Hello World";
    var password = "Pass";
    var maxtime = 1.0;

    var cipher = scrypt.encryptSync(message, password, maxtime);
    var plainText = scrypt.decryptSync(cipher, password, maxtime);

Note: that `maxmem` and `maxmemfrac` can also be passed to the functions. If they are not passed, then `maxmem` defaults to `0` and `maxmemfrac` defaults to `0.5`. If these values are to be passed, then they must be passed after `maxtime`  and before the callback function like so:
    
    var scrypt = require("scrypt");
    var message = "Hello World";
    var password = "Pass";
    var maxtime = 1.0;
    var maxmem = 1; //Defaults to 0 if not set
    var maxmemfrac = 1.5; //Defaults to 0.5 if not set

    var cipher = scrypt.encryptSync(message, password, maxtime, maxmem, maxmemfrac);
    var plainText = scrypt.decryptSync(cipher, password, maxtime, maxmem, maxmemfrac);

#Api

##Authentication

###Asynchronous
* `passwordHash(password, maxtime, maxmem, maxmemfrac, callback_function)`
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt. Use 0.1 (100 milliseconds) for interactive logins.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).
    * `callback_function` - [REQUIRED] - a callback function that will handle processing when result is ready.
* `verifyHash(hash, password, callback_function)` 
    * `hash` - [REQUIRED] - the password created with the above `passwordHash` function.
    * `password` - [REQUIRED] - a password string.
    * `callback_function` - [REQUIRED] - a callback function that will handle processing when result is ready.

###Synchronous
* `passwordHashSync(password, maxtime, maxmem, maxmemfrac)`
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt. Use 0.1 (100 milliseconds) for interactive logins.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).
* `verifyHashSync(hash, password)`
    * `hash` - [REQUIRED] - the password created with the above `passwordHash` function.
    * `password` - [REQUIRED] - a password string.
           
##Encryption/Decryption

###Asynchronous
* `encrypt(message, password, maxtime, maxmem, maxmemfrac, callback_function)`
    * `message` - [REQUIRED] - the message data to be encrypted.
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).
    * `callback_function` - [REQUIRED] - a callback function that will handle processing when result is ready.
* `decrypt(cipher, password, maxtime, maxmem, maxmemfrac, callback_function)`
    * `cipher` - [REQUIRED] - the cipher to be decrypted.
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).
    * `callback_function` - [REQUIRED] - a callback function that will handle processing when result is ready.

###Synchronous
* `encryptSync(message, password, maxtime, maxmem, maxmemfrac)`
    * `message` - [REQUIRED] - the message data to be encrypted.
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).
* `decryptSync(cipher, password, maxtime, maxmem, maxmemfrac)`
    * `cipher` - [REQUIRED] - the cipher to be decrypted.
    * `password` - [REQUIRED] - a password string.
    * `maxtime` - [REQUIRED] - a decimal (double) representing the maxtime in seconds for running Scrypt.
    * `maxmem` - [OPTIONAL] - instructs Scrypt to use the specified number of bytes of RAM (default 0).
    * `maxmemfrac` - [OPTIONAL] - instructs Scrypt to use the specified fracion of RAM (defaults 0.5).

#Credits
The Scrypt library is Colin Percival's [Scrypt](http://www.tarsnap.com/scrypt.html) project. This includes the encryption/decryption functions which are basically just wrappers into this library.

The password hash and verify functions are also very heavily influenced by the Scrypt source code, with most functionality being copied from various placed within Scrypt.

#Contributors

* [René Nyffenegger](http://www.adp-gmbh.ch/) - produced original Base64 encoding code.
* [Kelvin Wong](https://github.com/kelvinwong-ca) - MAC OS compilation and testing.
* [Tamas Geschitz](https://github.com/gtamas) - SmartOS and MAC OS testing
