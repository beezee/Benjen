#title Confusing Crypto Blobs
#date 2012-10-20

# 

# Introduction

In 2011, [CCBill][1] ran a bug bounty program. While they didn't pay much per-bug, there were enough low-hanging fruit available to make it fairly lucrative; you could find a couple bugs in an hour and make some quick money. So for a month or two, I'd dedicate one or two nights a week to hunting down bugs and reporting them. As you can see from the [rewards page][2] that worked out pretty well. However, one of the most interesting bugs I found isn't on there. This is the story of that bug.

 [1]: http://ccbill.com/
 [2]: http://www.ccbill.com/developers/security/rewards.php

# Interest

Early on in the bug bounty, one of my friends (Neal Poole, also testing CCBill at the time) pointed me in the direction of the 'Forgot Password' function and thought I might be able to find something there, specifically a padding oracle type attack. When I requested a password reset for my account, I'd receive a long link in my email, e.g.:

    https://admin.ccbill.com/updatePassword2.cgi?enc=c8d2d3
    d66ac13a865d64276623d8260352616e646f6d49566e31e063163e5
    bd501fbca5d6f1c71998148a12edb637a2cf36492b646d8ad9ab811
    9c6214432afe27c3c3030c2e6b43a2a8e9452923ad18298f9255578
    d639c07afe41c1e82935d1cdba3709bec565e3eefb08ad27f33127a
    d3daeb51d88d16209c706de526fc1c63f9e760e39da6fecbc9b5734
    a8bb8a8348a07c2aac7645a799586a0f4cac7fa075862f4cac6dc17
    137ad3840d43ea1d7a04d324cff6aab1917b4ac3779a98f3fdb8476
    f25f350ae9bc4863fa1e11b4eacfba44b4dded55e6589568b576758
    90635da75c13e22234

Now, when I see a blob like that, my first step is to take it from hex back to an ASCII representation. In the header of that blob, I saw `RandomIV`. A quick Googling of that string told me that it was generated by Perl's Crypt::CBC. Outside of an irrelevant old vulnerability report about the usefulness of the IVs ([here][3]), it looked solid. CBC mode, HMACed properly (seemingly), etc. So tampering with the blob wasn't much use.

 [3]: http://cpansearch.perl.org/src/LDS/Crypt-CBC-2.30/Crypt-CBC-2.16-vulnerability.txt

I gave up after trying a couple different attack routes on it, as it seemed outside my cryptographical expertise.

# Obsession

Of course, giving up and forgetting about something are two very different things, especially for me. Something about those blobs didn't seem right, and I started seeing them in other places. For instance, requesting reports about your account would lead you to a viewer that took another -- much larger -- encrypted blob, presumably containing either a query for the report data or the data itself.

So one night I found myself laying in bed and thinking about this issue, and whether I had missed something. I'd tried everything I could think of to break the blobs themselves, but there was some idea in the back of my mind.

# Source confusion

They use the same blobs in many places. What if they're using the same key in each place? If they are, then a blob from one place will decrypt properly in another. If you could control what goes into the blob, you could compromise the other components. He who controls the source, controls the universe.

I immediately got up and took one of my report tokens and put it into the forgot password page. Up popped up a password reset dialog.

Of course, I had no control yet -- I had no idea what was in the blob, how it was constructed, or how any of the pages actually used the data in the blob -- but that proved to me that they used the same key in each place, making this very likely exploitable.

# Gaining control

There was one place I could control the data going into the blob: report generation. While I couldn't ever tell just how much control I had over the final blob, I was able to put *some* arbitrary content into it, and that was good enough.

However, another interesting problem popped up soon after I discovered that: they were compressing the data going into the blob. If I put in a string of 'A's, I would only add a byte or two to the total size. That makes it harder to gain total control, but if you can put in any data you want, you could poison the compression enough to put it into a bad state -- one where effectively nothing compresses properly, and you end up with your own data completely in the clear.

The compression also could be used to figure out what else is in the blob *other* than your data. I didn't go down that rabbit hole, but this isn't unprecedented; in fact, it's the same way [CRIME works][4].

 [4]: http://security.stackexchange.com/a/19914/7102

# Impact

Through trial and error, you can determine what impact your data has on the page where you're using the blob. This is an odd one in that depending on what's done with the bug, it could be anywhere from useless to super critical. In this case, it was determined by CCBill's security staff that it was worth two high bugs, or $1000. I never did find out what exactly could have been done with this bug, but I do know that you could compromise the forgot password form, and cause some query failures with the reporting facility.

# Conclusion

If I were an attacker, this could've been a big, big win; in theory, this could've given me the keys to the castle. However, as a bounty participant it was worthless outside of scratching my own itch, as I took probably 5 or 6 hours to find that $1000 bug, rather than spending that same time to find ten $300 bugs. That said, it was definitely a more interesting result than the others.

## Advice for developers/security folks

These days it's well known how to get encrypted blobs right (even if most people don't): use sane crypto in a good mode with a random IV, HMAC appropriately to prevent tampering, and don't leak information to enable oracles. However, even if your blobs are perfectly sound, using them in multiple places with the same key makes it possible for attackers to use a blob from page A on page B instead. If the attacker can control the data inside the blobs *at all*, you're likely to be compromised. The solution is to use different keys for each page that uses a different blob, or to put a 'blob type' flag at the beginning of the plaintext data.

In most cases this will not be a good way to attack a site -- there's generally something juicier and easier -- but they're really easy to miss and can be critical.

## Advice for bug bounty admins

When determining pricing for bugs, make sure you aren't creating perverse incentives. A SQL injection vulnerability could cost you far more than a CSRF bug in a random user-facing page -- price accordingly. CCBill priced theirs at $300 for low, $400 for medium, and $500 for high bugs, though both Neal and I received one or two 2*high payouts. If you can find three or four low bugs ($900-1200) in the time it takes you to find a single high bug ($500), it absolutely makes sense to focus on those low-hanging fruit; the pricing is simply broken in that case. Not that I mind.

Happy Hacking,   
- Cody Brocious (Daeken)

## P.S.

Want me to find bugs like this in your own products? Check out [my portfolio][5] and [contact me](mailto:cody.brocious+work@gmail.com) about my consulting services.

 [5]: http://demoseen.com/portfolio/