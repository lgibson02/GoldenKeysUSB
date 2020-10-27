# Golden Keys Original Writeup
Originally From: https://rol.im/securegoldenkeyboot/

```
                |                                             |
                | a  w r i t e u p  r e l e a s e  b y  r o l |
                |      ________  ___  ________  ________      |
                |     <_  __   \/   \/        \/ ____   \     |                  
                |      T  T<___/\___/\_  /\  _/\ \__j  _/     |
                |      |  |     T   T T /  \ T__\____  T      |
                |      |  |     |   | | \  / |T T   T  |      |
                |      l__j_____l___j_l__><__j| |   |  |      |
                |       T _______ T | ___j    | l___j  |      |
                |       | T   __T |_j l_______l________j      |
                |       | |   l_| |__ _______j                |
                |       | l_____j | T T                       |
           ____ '     __l_________j_| |___                    `  ________
           T  T  ___ / ____  TT  __Tj |  T     _/\_   ____/\_   / ____  T
           |  | /   \\ \__j  ||  l____j  |   _/    \_/  \    \_ \ \__j  |
           |  |_\___/_\____  ||    l__|  l___T  /\  T___/ /\  T__\____  |
           |  | TT  T T   T  ||   _   |  ___j| /  \ |  T /  \ |T T   T  |
           |  l ||  | l___j  ||   |   |  l___| \  / |  | \  / || l___j  |
           l____jl__l________jl___l___j______j__><__j__j__><__jl________j
            r   i   n   g     o   f     l   i   g   h   t   n   i   n   g

irc.rol.im #rtchurch :: https://rol.im/chat/rtchurch

Specific Secure Boot policies, when provisioned, allow for testsigning to be
enabled, on any BCD object, including {bootmgr}. This also removes the NT loader
options blacklist (AFAIK). (MS16-094 / CVE-2016-3287, and MS16-100 / CVE-2016-3320)

Found by my123 (@never_released) and slipstream (@TheWack0lian)
Writeup by slipstream (@TheWack0lian)

First up, "Secure Boot policies". What are they exactly?

As you know, secureboot is a part of the uefi firmware, when enabled, it only
lets stuff run that's signed by a cert in db, and whose hash is not in dbx
(revoked).

As you probably also know, there are devices where secure boot can NOT be
disabled by the user (Windows RT, HoloLens, Windows Phone, maybe Surface Hub,
and maybe some IoTCore devices if such things actually exist -- not talking
about the boards themselves which are not locked down at all by default, but end
devices sold that may have secureboot locked on).

But in some cases, the "shape" of secure boot needs to change a bit. For example
in development, engineering, refurbishment, running flightsigned stuff (as of
win10) etc. How to do that, with devices where secure boot is locked on?

Enter the Secure Boot policy.

It's a file in a binary format that's embedded within an ASN.1 blob, that is
signed. It's loaded by bootmgr REALLY early into the windows boot process. It
must be signed by a certificate in db. It gets loaded from a UEFI variable in
the secureboot namespace (therefore, it can only be touched by boot services).
There's a couple .efis signed by MS that can provision such a policy, that is,
set the UEFI variable with its contents being the policy.

What can policies do, you ask?

They have two different types of rules. BCD rules, which override settings
in the on-disk BCD, and registry rules, which contain configuration for the
policy itself, plus configuration for other parts of boot services, etc. For
example, one registry element was introduced in Windows 10 version 1607
'Redstone' which disables certificate expiry checking inside mobilestartup's
.ffu flashing (ie, the "lightning bolt" windows phone flasher); and another one
enables mobilestartup's USB mass storage mode. Other interesting registry
rules change the shape of Code Integrity, ie, for a certain type of binary,
it changes the certificates considered valid for that specific binary.

(Alex Ionescu wrote a blog post that touches on Secure Boot policies. He teased a
followup post that would be all about them, but that never came.)

But, they must be signed by a cert in db. That is to say, Microsoft.

Also, there is such a thing called DeviceID. It's the first 64 bits of a salted
SHA-256 hash, of some UEFI PRNG output. It's used when applying policies on
Windows Phone, and on Windows RT (mobilestartup sets it on Phone, and
SecureBootDebug.efi when that's launched for the first time on RT). On Phone,
the policy must be located in a specific place on EFIESP partition with the
filename including the hex-form of the DeviceID. (With Redstone, this got
changed to UnlockID, which is set by bootmgr, and is just the raw UEFI PRNG
output.)

Basically, bootmgr checks the policy when it loads, if it includes a DeviceID,
which doesn't match the DeviceID of the device that bootmgr is running on, the
policy will fail to load.

Any policy that allows for enabling testsigning (MS calls these Retail Device
Unlock / RDU policies, and to install them is unlocking a device), is supposed
to be locked to a DeviceID (UnlockID on Redstone and above). Indeed, I have
several policies (signed by the Windows Phone production certificate) like
this, where the only differences are the included DeviceID, and the signature.

If there is no valid policy installed, bootmgr falls back to using a default
policy located in its resources. This policy is the one which blocks enabling
testsigning, etc, using BCD rules.

Now, for Microsoft's screwups.

During the development of Windows 10 v1607 'Redstone', MS added a new type of
secure boot policy. Namely, "supplemental" policies that are located in the
EFIESP partition (rather than in a UEFI variable), and have their settings
merged in, dependant on conditions (namely, that a certain "activation" policy
is also in existance, and has been loaded in).

Redstone's bootmgr.efi loads "legacy" policies (namely, a policy from UEFI
variables) first. At a certain time in redstone dev, it did not do any further
checks beyond signature / deviceID checks. (This has now changed, but see how
the change is stupid)
After loading the "legacy" policy, or a base policy from EFIESP partition, it
then loads, checks and merges in the supplemental policies.

See the issue here? If not, let me spell it out to you plain and clear.
The "supplemental" policy contains new elements, for the merging conditions.
These conditions are (well, at one time) unchecked by bootmgr when loading a
legacy policy. And bootmgr of win10 v1511 and earlier certainly doesn't know
about them. To those bootmgrs, it has just loaded in a perfectly valid, signed
policy.

The "supplemental" policy does NOT contain a DeviceID. And, because they were
meant to be merged into a base policy, they don't contain any BCD rules either,
which means that if they are loaded, you can enable testsigning. Not just for
windows (to load unsigned driver, ie rootkit), but for the {bootmgr} element
as well, which allows bootmgr to run what is effectively an unsigned .efi
(ie bootkit)!!! (In practise, the .efi file must be signed, but it can be
self-signed) You can see how this is very bad!! A backdoor, which MS put 
in to secure boot because they decided to not let the user turn it off in
certain devices, allows for secure boot to be disabled everywhere!

You can see the irony. Also the irony in that MS themselves provided us several
nice "golden keys" (as the FBI would say ;) for us to use for that purpose :)

About the FBI: are you reading this? If you are, then this is a perfect real
world example about why your idea of backdooring cryptosystems with a "secure
golden key" is very bad! Smarter people than me have been telling this to you
for so long, it seems you have your fingers in your ears. You seriously don't
understand still? Microsoft implemented a "secure golden key" system. And the
golden keys got released from MS own stupidity. Now, what happens if you tell
everyone to make a "secure golden key" system? Hopefully you can add 2+2...

Anyway, enough about that little rant, wanted to add that to a writeup ever
since this stuff was found ;)

Anyway, MS's first patch attempt. I say "attempt" because it surely doesn't do
anything useful. It blacklists (in boot.stl), most (not all!) of the policies.
Now, about boot.stl. It's a file that gets cloned to a UEFI variable only boot
services can touch, and only when the boot.stl signing time is later than the
time this UEFI variable was set.
However, this is done AFTER a secure boot policy gets loaded. Redstone's
bootmgr has extra code to use the boot.stl in the UEFI variable to check
policy revocation, but the bootmgrs of TH2 and earlier does NOT have such
code.
So, an attacker can just replace a later bootmgr with an earlier one.

Another thing: I saw some additional code in the load-legacy-policy function in
redstone 14381.rs1_release. Code that wasn't there in 14361. Code that
specifically checked the policy being loaded for an element that meant this was
a supplemental policy, and erroring out if so. So, if a system is running
Windows 10 version 1607 or above, an attacker MUST replace bootmgr with
an earlier one.

On August 9th, 2016, another patch came about, this one was given the designation
MS16-100 and CVE-2016-3320. This one updates dbx. The advisory says it revokes
bootmgrs. The dbx update seems to add these SHA256 hashes (unless I screwed up
my parsing):

075eea060589548ba060b2feed10da3c20c7fe9b17cd026b94e8a683b8115238
07e6c6a858646fb1efc67903fe28b116011f2367fe92e6be2b36999eff39d09e
09df5f4e511208ec78b96d12d08125fdb603868de39f6f72927852599b659c26
0bbb4392daac7ab89b30a4ac657531b97bfaab04f90b0dafe5f9b6eb90a06374
0c189339762df336ab3dd006a463df715a39cfb0f492465c600e6c6bd7bd898c
0d0dbeca6f29eca06f331a7d72e4884b12097fb348983a2a14a0d73f4f10140f
0dc9f3fb99962148c3ca833632758d3ed4fc8d0b0007b95b31e6528f2acd5bfc
106faceacfecfd4e303b74f480a08098e2d0802b936f8ec774ce21f31686689c
174e3a0b5b43c6a607bbd3404f05341e3dcf396267ce94f8b50e2e23a9da920c
18333429ff0562ed9f97033e1148dceee52dbe2e496d5410b5cfd6c864d2d10f
2b99cf26422e92fe365fbf4bc30d27086c9ee14b7a6fff44fb2f6b9001699939
2bbf2ca7b8f1d91f27ee52b6fb2a5dd049b85a2b9b529c5d6662068104b055f8
2c73d93325ba6dcbe589d4a4c63c5b935559ef92fbf050ed50c4e2085206f17d
2e70916786a6f773511fa7181fab0f1d70b557c6322ea923b2a8d3b92b51af7d
306628fa5477305728ba4a467de7d0387a54f569d3769fce5e75ec89d28d1593
3608edbaf5ad0f41a414a1777abf2faf5e670334675ec3995e6935829e0caad2
3841d221368d1583d75c0a02e62160394d6c4e0a6760b6f607b90362bc855b02
3fce9b9fdf3ef09d5452b0f95ee481c2b7f06d743a737971558e70136ace3e73
4397daca839e7f63077cb50c92df43bc2d2fb2a8f59f26fc7a0e4bd4d9751692
47cc086127e2069a86e03a6bef2cd410f8c55a6d6bdb362168c31b2ce32a5adf
518831fe7382b514d03e15c621228b8ab65479bd0cbfa3c5c1d0f48d9c306135
5ae949ea8855eb93e439dbc65bda2e42852c2fdf6789fa146736e3c3410f2b5c
6b1d138078e4418aa68deb7bb35e066092cf479eeb8ce4cd12e7d072ccb42f66
6c8854478dd559e29351b826c06cb8bfef2b94ad3538358772d193f82ed1ca11
6f1428ff71c9db0ed5af1f2e7bbfcbab647cc265ddf5b293cdb626f50a3a785e
71f2906fd222497e54a34662ab2497fcc81020770ff51368e9e3d9bfcbfd6375
726b3eb654046a30f3f83d9b96ce03f670e9a806d1708a0371e62dc49d2c23c1
72e0bd1867cf5d9d56ab158adf3bddbc82bf32a8d8aa1d8c5e2f6df29428d6d8
7827af99362cfaf0717dade4b1bfe0438ad171c15addc248b75bf8caa44bb2c5
81a8b965bb84d3876b9429a95481cc955318cfaa1412d808c8a33bfd33fff0e4
82db3bceb4f60843ce9d97c3d187cd9b5941cd3de8100e586f2bda5637575f67
895a9785f617ca1d7ed44fc1a1470b71f3f1223862d9ff9dcc3ae2df92163daf
8ad64859f195b5f58dafaa940b6a6167acd67a886e8f469364177221c55945b9
8bf434b49e00ccf71502a2cd900865cb01ec3b3da03c35be505fdf7bd563f521
8d8ea289cfe70a1c07ab7365cb28ee51edd33cf2506de888fbadd60ebf80481c
9998d363c491be16bd74ba10b94d9291001611736fdca643a36664bc0f315a42
9e4a69173161682e55fde8fef560eb88ec1ffedcaf04001f66c0caf707b2b734
a6b5151f3655d3a2af0d472759796be4a4200e5495a7d869754c4848857408a7
a7f32f508d4eb0fead9a087ef94ed1ba0aec5de6f7ef6ff0a62b93bedf5d458d
ad6826e1946d26d3eaf3685c88d97d85de3b4dcb3d0ee2ae81c70560d13c5720
aeebae3151271273ed95aa2e671139ed31a98567303a332298f83709a9d55aa1
afe2030afb7d2cda13f9fa333a02e34f6751afec11b010dbcd441fdf4c4002b3
b54f1ee636631fad68058d3b0937031ac1b90ccb17062a391cca68afdbe40d55
b8f078d983a24ac433216393883514cd932c33af18e7dd70884c8235f4275736
b97a0889059c035ff1d54b6db53b11b9766668d9f955247c028b2837d7a04cd9
bc87a668e81966489cb508ee805183c19e6acd24cf17799ca062d2e384da0ea7
c409bdac4775add8db92aa22b5b718fb8c94a1462c1fe9a416b95d8a3388c2fc
c617c1a8b1ee2a811c28b5a81b4c83d7c98b5b0c27281d610207ebe692c2967f
c90f336617b8e7f983975413c997f10b73eb267fd8a10cb9e3bdbfc667abdb8b
cb6b858b40d3a098765815b592c1514a49604fafd60819da88d7a76e9778fef7
ce3bfabe59d67ce8ac8dfd4a16f7c43ef9c224513fbc655957d735fa29f540ce
d8cbeb9735f5672b367e4f96cdc74969615d17074ae96c724d42ce0216f8f3fa
e92c22eb3b5642d65c1ec2caf247d2594738eebb7fb3841a44956f59e2b0d1fa
fddd6e3d29ea84c7743dad4a1bdbc700b5fec1b391f932409086acc71dd6dbd8
fe63a84f782cc9d3fcf2ccf9fc11fbd03760878758d26285ed12669bdc6e6d01
fecfb232d12e994b6d485d2c7167728aa5525984ad5ca61e7516221f079a1436
ca171d614a8d7e121c93948cd0fe55d39981f9d11aa96e03450a415227c2c65b
55b99b0de53dbcfe485aa9c737cf3fb616ef3d91fab599aa7cab19eda763b5ba
77dd190fa30d88ff5e3b011a0ae61e6209780c130b535ecb87e6f0888a0b6b2f
c83cb13922ad99f560744675dd37cc94dcad5a1fcba6472fee341171d939e884
3b0287533e0cc3d0ec1aa823cbf0a941aad8721579d1c499802dd1c3a636b8a9
939aeef4f5fa51e23340c3f2e49048ce8872526afdf752c3a7f3a3f2bc9f6049
64575bd912789a2e14ad56f6341f52af6bf80cf94400785975e9f04e2d64d745
45c7c8ae750acfbb48fc37527d6412dd644daed8913ccd8a24c94d856967df8e

I checked the hash in the signature of several bootmgrs of several
architectures against this list, and found no matches. So either this
revokes many "obscure" bootmgrs and bootmgfws, or I'm checking the wrong hash.

Either way, it'd be impossible in practise for MS to revoke every bootmgr
earlier than a certain point, as they'd break install media, recovery partitions,
backups, etc.

- RoL

disclosure timeline:
~march-april 2016 - found initial policy, contacted MSRC
~april 2016 - MSRC reply: wontfix, started analysis and reversing, working on
  almost-silent (3 reboots needed) PoC for possible emfcamp demonstration
~june-july 2016 - MSRC reply again, finally realising: bug bounty awarded
july 2016 - initial fix - fix analysed, deemed inadequate. reversed later rs1
  bootmgr, noticed additional inadequate mitigation
august 2016 - mini-talk about the issue at emfcamp, second fix, full writeup
  release


credits:
my123 (@never_released) -- found initial policy set, tested on surface rt
slipstream (@TheWack0lian) -- analysis of policies, reversing bootmgr/
  mobilestartup/etc, found even more policies, this writeup.

tiny-tro credits:
code and design: slipstream/RoL
font: dMG/Up Rough & Divine Stylers
awesome chiptune: bzl/cRO <3
```