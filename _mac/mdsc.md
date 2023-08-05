---
layout: rfc
title: MAC Demo Sharing and Coordination Protocol
lead: A DRAFT revision of Lilith's MDSCP proposal
type: DRAFT RFC
toc: true
---

# RFC MegaAntiCheat Client-Server Communication and Standards Protocol

This document proposes a protocol called 'The MAC Demo Sharing and Coordination Protocol' or 'MDSC Protocol' for short. 

**Document Version**: v0.0.1  
**Document Date**: 04/08/2023

**Authors**

<!-- 
|   author name    | official contact method | alternative contact methods  |
-->

|      Name        |     Contact             |   Alternate Contact          |
|:----------------:|:-----------------------:|:----------------------------:|
| Lilith           | Discord: __lilith       | [https://lilithwolf.vip](https://lilithwolf.vip)       |
|=====
{: .formal_table}

This document is intended for use and distribution in relation to the MegaAntiCheat project, managed by megascatterbomb.

Links:
- [Discord](https://discord.gg/megascatterbomb)
- [GitHub](https://github.com/MegaAntiCheat)

### LICENSE

This document is published under the [CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/) license.

Definition:
- Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.
- NoDerivatives — If you remix, transform, or build upon the material, you may not distribute the modified material.

Any recommendations, proposals, modifications, or corrections must be made through the official channels and merged into the document officially.

Note that the MAC source code that is available through its public GitHub repositories are GPLv3 licensed, and any code blocks or code examples provided in this document are also provided under the GPLv3 license. This does not extend to any of the surround media or material.

---

## Foreword

This is a _DRAFT_ revision of a formal proposal and request for comments (RFC) on proposing a MAC Client (MC) and MasterBase (MB) bidirectional communication protocol that focuses on verifying honesty and coordinating a distributed group of clients, robust to a given size population of dishonest users without impacting the experience of honest users. 

## What does this mean?

It means:
- MB poses as an inherently trusted service, verifiable on the client side using public key cryptography and message signatures
- MB implicitly distrusts every user by default, except for users labelled as 'MAC Admin' or 'Trusted', which are implicitly trusted over everyone else. 
- MB seeks to verify the data sent via Weighted Robust Consensus Agreement, a commonly used tactic in distributed untrusted systems.
  - This means that:
    - Certain components of the data that MC's sent to the MB must corroborate data sent by other MCs.
    - If data is not corroborated by enough users, it is dropped.
    - Resilient up to `N`% total dishonest users
    - Consensus can be weighted to value the input of certain users more so than others, which allows...
- Users are allocated a dynamic trust factor, where:
  - Normal/expected MAC behaviour gets 'rewarded' with positive increments to the trust factor.
  - Abnormal, malicious, dishonest or deceitful behaviour gets punished with heavy reductions to trust factor.
  - A higher trust factor correlates to your data having a higher weighting in consensus agreements.

---
{% include img-full.html path="assets/png/megascatterbomb_better.png" alt="MSB banner created by @dangerousdpad on Discord." %}
> Image credit: created by @dangerousdpad on Discord

# Context

MegaAntiCheat is an open-source community funded project to create a third-party tool to monitor, track and convict players scripting and cheating in the hit 2007 team-based multiplayer first-person shooter Team Fortress 2 created and distributed by Valve Corporation. 

On April 22nd, 2020, the source code of many source games, including Team Fortress 2, was leaked. This source code can be found hosted on GitHub in several places, including [here](https://github.com/lua9520/source-engine-2018-hl2_src). In combination with this and some long-time script 'developers', more and more cheats flooded the TF2 community, creating the environment we see today: Cheating bot players that automate join servers and attempting to ruin them, large numbers of 'closet' cheaters who use a selection of paid and open-source TF2 cheats to silently boost their skill, and large numbers of 'blatant' cheaters who use the same cheats to ruin the game for others.

Before the MegaAntiCheat project, YouTuber and Oceanic-based TF2 player [megascatterbomb](https://www.youtube.com/@megascatterbomb) had created (and eventually publically published) a database for tracking cheaters, bots and suspected 'closet' cheaters he and his friends had encountered in games. The biggest problem with this project was the lack of evidence and verifiability for why players were or weren't marked on this database. 

This lead to the creation of the MegaAntiCheat project - Team Fortress 2 Demos (more on these later) can be submitted, automatically analysed by machine learning models and greatly assist in the labelling of cheaters and non-cheaters. Since this tool works off of TF2 Demo files, the evidence as to a player conviction will always be publically available and verifiable. MegaAntiCheat will be managed on the users side with a local client that will facilitate and automate a lot of actions, including communicating with the MegaAntiCheat servers (called 'MasterBase') and actioning requests from the MasterBase such as to kick a convicted player from the current match.

### Team Fortress 2 and Demos

Team Fortress 2 features an official match making service, that puts you in unlisted official Valve-owned and operated servers. Team Fortress 2 provides a 'voting' interface for players in game, that is primarily instituted for voting to kick a player on your team. This function is the primary tool that players use to remove cheaters and bots from their lobbies, but requires someone on the team of the cheater/bot to initiate the vote, and only players on that given team can vote. Vote-kicks also required a 60% yes ratio to be successful, and since official matchmaking supports parties, sometimes there are too many cheaters or dishonest players in a match for the honest players to kick. This is a known and unsolvable problem, which is typically avoided simply by 'jumping lobbies' or re-queuing for to be put in another server.

Team Fortress 2 allows players to record official 'demos' of their games, which simply acts as a file that captures and contains all network packets that jump between you and the game server throughout the course of a game. This file can then be later played back, and displays and exact replica of that game, played from the perspective of the game and server that the recording client had.

These demo files can be converted into 'replays', that allow you to 'spectate' from the perspective of any player in the lobby, as the data regarding that player was received by the recording client.

As these demo and replay files contain the exact packets sent and received by the client to and from that given game server, they serve as a good base representation of a game, and can put first-hand data to certain actions, such as the exact position and view-angle of a given player from one server 'tick' to another.

### How do we intend to interface and work with Team Fortress 2?

Team Fortress 2, being a game built on the Source 1 engine, has quite a sophisticated developer console and tooling. The developer console supports a 'remote console' (RCon) that allows an external tool to send commands and receive responses from the TF2 console. TF2 also supports outputting the console data to a log file, which we can interactively read and interpret to maintain a perspective of the game and client state from outside the game.

This remote console can be used to automate tasks such as initiating or voting in player vote-kicks, grabbing a list of players in the current lobby by unique identifier, and more.

These unique identifiers ('Steam IDs') can be used to create a bijection between a given player in a TF2 Lobby, and a Steam account. By tracking the conviction status and history of players via their unique, fixed, Steam ID, we can maintain a persistent and permanent record of all players that the MasterBase sees.

Team Fortress 2 game servers host an API endpoint that allows any client to request a list of all players (by nickname, not Steam ID) in the current lobby, as well as some other scoreboard statistics. The MasterBase can use this to verify certain data sets submitted by clients.

### What are our restrictions?

As we are an honest and open source community intending to create an Anti-Cheat software, we refuse to incorporate techniques or tools used by the cheating community to interface directly with TF2 or the players machine. As such, we will never:
- Inject libraries or code into the Team Fortress 2 memory space
- Read the Team Fortress 2 memory space, or any program or system memory that isn't owned by our client application
- 'Sniff' or intercept any network packets.
- Monitor, watch or spy on the system network interfaces

More generally, any tools or clients we make for the purpose of this project MUST exist standalone and externally to Team Fortress 2, and can ONLY use information that the game publishes and is accessible to any honest user regularly.

Additionally, `admin` or `sudo`/`root` permissions will **never** be required to run our application, and as such our application will **never perform any privileged system operations**.

### What are the known problems?

MegaAntiCheat, being a fully open-source and freely available software, means that dishonest users can freely modify our client tools (or create their own) to submit junk or manipulated data to the MasterBase.

TF2 Demos are client controlled and recorded data, in which the MasterBase cannot verify remains unmodified. For example, there is no way for the MasterBase to confirm that a submitted demo file hasn't had one player swapped for another one, that potentially wasn't even in the given match. 

We cannot reasonably assume that any notable majority of TF2 players will be honest users and submitted unmodified data, and as such, we cannot assume any players are inherently honest.

TF2 Demo files are only written to disk in 576kb chunks. On a normal, full, official matchmaking server, this will occur ever 20-30 seconds on average. Smaller or more empty servers will have a larger time gap between disk writes.

TF2 Demo's cause a small to moderate lag spike upon starting, making constantly starting and stopping demo's a terrible user experience.

The TF2 console does not log all events that occur in a server. The TF2 console will let you call a vote on a given player, but to vote on an existing issue that another player has initiated requires the ID of the vote. The vote ID is not published in the console and there is no known command to extract it via the console and/or RCon.

The vote events and vote ID are tracked in the demo file, as they are events propagated by network packet.

TF2 votes initiated by a player last for 15 seconds. This means a vote can fall entirely between to demo disk writes and be completely missed by the client and/or the MasterBase. If we intend to automate calling and voting in player vote-kicks, we need to find a way to more routinely extract demo data.

---
{% include img-full.html path="assets/jpg/mvm-wallpaper-reddit.jpg" alt="Created by /u/Gx4 on Reddit" %}
> Image credit: /u/Gx4 on Reddit, uploaded [here](https://www.reddit.com/r/tf2/comments/yby1m/i_made_a_mvm_wallpaper_1920x1080/)

## Let's Break Down the Problem

This communication protocol has several components to it, which can be considered as 'sub-protocols' under the root protocol. 

### Protocols
Each protocol has a technical name (and corresponding acronym or abbreviation), and a formal name. The technical name is meant to concisely convey the purpose and action of the protocol, and the acronym/abbreviation is intended to be used to refer to this protocol when choosing to use the technical name. The formal name is a more human-friendly and readable name that can be associated with an overarching intention.

> E.g. The Convict Detection and Consensus Agreement Coordination Response protocol, with acronym 'CDCACR Protocol', has formal name 'The Interceptor Protocol'.

This protocol could be referred to interchangeably as any of the 3 names, and in formal documentation would be referred to by full technical name AND formal name upon the first mention, and then from then onwards may be referred to by either the acronym OR the formal name interchangeably.   

### The 'Parent' Protocol
The above rules and style guides apply to the top-level 'parent' protocol as well, which has technical name 'The MAC Demo Sharing and Coordination Protocol', or 'MDSC Protocol', and formal name TBD (awaiting input from @megascatterbomb). The parent protocol will serve as a wrapper for a set of sub-protocols, and the requisite protocol definitions to safely and securely tie all sub-protocols together.

## But Wait, What If...

Here is some additional information to help alleviate and/or avoid a lot of repetitive 'What if...' or 'Why does...' questions:
- For an MC to send messages the MB, it must have an authenticated API key associated with the given steam account. This means:
  - All MC → MB communication is authenticated, signed, verified and encrypted.
  - No client _not_ in a given game can submit information regarding that game
  - We can proactively and retroactively ban users from interacting with MB in any meaningful way.
- The MB's messages will be signed by its private signing key, and communicated via TLS. This means:
  - All MB → MC communication is verifiable as to where it came from, and if it's been modified en route.
  - No reasonable attacker could impersonate the MB in a way that was not detectable by our provided client.
    - An externally distributed client could have modified MB signatures, which would allow a valid MITM attack.
    - Any attacker with privileged access to the Public Key Infrastructure (such as a federal government sponsored attacker) could still pose a threat. 
  - An attacker can not modify the en route data without leaving a detectable footprint (signature will no longer validate)
  - An attacker can not read data en route without knowing your systems private key and having full access to your communication history

As more questions are raised, they will be added to this section as the document is iterated. 

---
{% include img-full.html path="assets/jpg/markus-spiske-code.jpg" with_caption=true caption="MegaAntiCheat Protocol" subtext="The MAC Demo Sharing and Coordination Protocol" %}

# MAC Demo Sharing and Coordination Protocol

For a MAC Client (MC) to request entry into the MasterBase (MB) Lobby Coordination Pool (LCP), the MAC client must implement the MAC Demo Sharing and Coordination Protocol (MDSCP) and connect to MB using this protocol. For as long as an MC maintains an MDSCP connection with the MB, it may post data to the MB, and receive coordination requests from the MB. The MDSCP protocol is a 'proof of honesty' protocol that any honest MC must implement to provide a climate in which the MB can develop a trust factor with that client.

The MDSC Protocol enacts several sub-protocols to facilitate distributed consensus agreement on posted data, maintain honest communications and coordinate actions against detected convicted players. These are:

- **The Consensus Protocol**, A.K.A. The Sharing Agreeable Data protocol (SAD protocol) 
- **The Brick Wall Protocol**, A.K.A. The Continuous Demo Chunk Data Exchange and Synchronized Hash Check protocol (The CDCDESHC protocol)
- **The Interceptor Protocol**, A.K.A. The Convict Detection and Consensus Agreement Coordination Response protocol (The CDCACR protocol)
> Note that protocol formal names (the bold names) are subject to change at any time, regardless of document semantic versioning.

All data that a client wishes to send to the MB is coordinated via the Sharing Agreeable Data protocol (The SAD Protocol), which is the primary driver of trust factor change. SAD uses distributed consensus agreement to decide which datasets to trust as 'more representative of the base truth'. The higher your trust factor, the more weighted an MCs data. Higher trust factor implies the MB will trust your data more than other data. This is used to further weight the consensus agreements. Do note that a high trust factor does not inherently mean that the MB will always trust your data. Data provided by an MC with a high trust factor can still be rejected by the MB.

If an MC wishes to join the LCP and maintain an MDSCP connection, it must enact the CDCDESHC protocol. This involves the client sending demo data to the MB as it is created and dumped to disk. This data is transmitted via SAD and must share agreeable traits with other MCs in the same lobby. Joining the LCP works on an 'ephemeral lease' basis, with every demo chunk provided to the MB by the MC extending their lease. Failure to provide a new demo chunk without requesting leave of the LCP within a certain timeframe will result in that MC being dropped from the LCP and a trust factor reduction.

If an MC wishes to maintain their status in the LCP, they must enact the CDCACR protocol. This involves communicating with the MB when a labelled convict joins the lobby, and triggering demo splits to facilitate the MB-side coordination of a vote-kick. Failure for an MC to publish information to the MB regarding a present convict will result in the MB dropping that MC from the LCP and a trust factor reduction. Any demo data messages sent to MB will extend the LCP ephemeral lease as normal, but non-demo data messages will not contribute to the lease. (i.e. MCs must also continue standard behavior during this process)

## Protocol Components

### The baseline

With respect to the architecture of TF2 and its various game and game coordination services, we cannot trust _any_ data that a client chooses to send us. Further, the only data that the MB can validate is that a specified server is correct, and that the players on that server match. With this in mind, it becomes apparent that we are operating in a heavily distributed and untrusted environment. The general rules surrounding this are as such:

- We (the MB) cannot trust _any_ data provided by a client that we cannot personally validate.
- Demo data can be manipulated in any way and remain undetected unless it's a modification to the list of users present in that lobby.
- A demo, unless recorded by the server itself, will never convey the true game-state, and instead will present a client-biased representation of a sliding scale of proximity to the base truth.

We can orient and design our security environment in such a way that we can infer following: 

- We can _assume_ that, on average, there will be at least one honest MAC user in a given population.
- We can blindly trust any user that MB is labelled as a 'MAC Admin'
- We can assume with higher probability that data presented by 'trusted' accounts is not manipulated.

This allows us to enact the following policies:

- Any user presenting data directly disparate to a 'MAC Admin' labelled account is immediately untrusted
- Any user presenting data directly disparate to a 'trusted' account will suffer a trust factor penalty
- User trust factors are used to weight the 'trustworthiness' and importance of the data they present.
- All decisions that the MB chooses to make, or actions that it takes that rely on user presented data must be provided via the Sharing Agreeable Data Protocol (SAD Protocol) 

This allows us to infer the following regarding the trustworthiness of given clients:

- Any user that routinely agrees with a highly trusted account will be considered more trustworthy
- Any user that routinely agrees with the majority in their lobbies, and at least one user of that majority has a high trust factor, or the 'trusted' or 'MAC Admin' label, will be considered more trustworthy
- Any user that reports information and responds to coordination requests that result in a verifiable and correct action being taken will receive be considered more trustworthy  
- Any user that routinely disagrees with the majority, or with an account with a suitably higher trust factor, or a 'MAC Admin'/'trusted' label, will be considered dishonest (less trustworthy)
- Any user that reports verifiably false or misleading information will be considered dishonest (less trustworthy)
- Any user that is labelled as 'trusted' or has a high trust factor, and disagrees with an account labelled as 'MAC Admin' will have their trust factor reset and the 'trusted' label potentially stripped.
- Any user that is labelled as 'trusted' or has a high trust factor, and falls on the minority side of a consensus agreement a given number of times in a given timeframe will suffer a trust factor reduction and potential loss of the 'trusted' label.
    
**Notes**

A 'MAC Admin' labelled user is a user that is highly trusted and will always present unmanipulated, correct, information. 'MAC Admin' labels are STRICTLY controlled MasterBase server-side tags that are not communicated to other MCs ever unless that other MC is also labelled as a 'MAC Admin'.

### SAD Protocol (The Consensus Protocol)

Sharing Agreeable Data Protocol

When we refer to a user providing or sharing 'data' with MB, we are referring to a complete, or chunk of a complete, Demo file.

The MasterBase may trust user provided data when it meets the all the following conditions:
- The data is provided by a user with a non-negative trust factor
- The data is provided by a non-convicted user
- The data meets any of the following consensus policies:
    * Is presented by, or corroborates data provided by, a 'MAC Admin' labelled user.
    * Corroborates with, after adjustments for trust-factor ratings, at least 61% of the participating MCs in this LCP.
    * Corroborates with, after adjustments for trust-factor ratings, at least 6, or 90% (which ever is greater) of the participating MCs on the clients team, and the lobby is an official matchmaking lobby.

For data to 'corroborate' or 'agree' with another MC, some but not all values contained in the message must match. Which data this is may be dependent on the parent protocol in which SAD is being used. For example:
- When reporting a convicted player joining the game, the lobby players, server information and map must match.
- When reporting a vote-kick initialization, the lobby players, the target of the current vote, the player who called the vote, the vote reason, server information and map must match.

Generally, for a vote-kick to be successful in TF2, it must receive 60% yes votes.

Whenever an MC provides data that meets the above conditions, that MCs trust factor is increased based on their previous history and trust factor.

**Reasoning/Notes**

Some community servers may allow players on the other team to vote on vote-kicks, which is why the third consensus policy is only available on official match-made servers, which lock voting to the relevant team only.

Since vote-kicks on official match-made servers can only affect players on the same team as the MC, if dishonest/nefarious users already represent the majority of players on that team, they already hold the vote-kick advantage on any player they want. Thus, we can short-circuit data used for coordinating the Interceptor protocol on official match-made servers to be trusted if the majority of players on a given team, rather than the whole game, agree with it.

### CDCDESHC Protocol (The Brick Wall Protocol)

The Continuous Demo Chunk Data Exchange and Synchronized Hash Check protocol

**Background**

Demo recordings in TF2 are essentially a broad-spectrum capture of all sent and received network packets, recorded into an internal buffer. When this internal buffer fills up, it will dump the data to the demo file and clear the buffer, repeating the process. In your average 12v12 Casual match, this occurs every 20 to 30 seconds and every 576kb. We can primitively approach these 576kb blocks of a demo file as 'Demo Chunks'.

**Protocol**

For an MC to be allowed to continue an active connection with the MB, they must implement and enact the CDCDESHC protocol, which has the following requirements:

- Every time the demo file in the file system is updated with the next Demo Chunk:
    * that client should send a Standard Demo Chunk Message (SDCM) abiding by all rules
    * the MB compares the message send and receive time with previous SDCM's sent by this MC
      * If the Demo Chunk contains less information than we know to expect, the MC's MDSCP connection is dropped and trust factor reduced. 
      * If the provided system time and demo tick time vary greatly from the expected values
        * The relevant MC will suffer a trust factor reduction
    * the MB compares this message with messages from other MCs in the same lobby
      * If the message contains a demo chunk that is identical between two or more MCs:
        * All relevant MCs will suffer a trust factor reduction
    * the MB compares this demo chunk hash with a library of known demo chunk hashes received previously by the same and other MCs from other games of the same metadata
      * If the demo chunk shares an exact hash with a demo chunk of a previous lobby, it is considered duplicated or replayed, and dropped, and the sending clients MDSCP connection is dropped. The expected trust factor reductions occur.
    * The MB will respond with a hash of that MCs 'game-to-date' demo, which should match exactly the hash of the local clients `.dem` file. Receiving this hash should be considered as a positive acknowledgement of receipt of demo data.
- Whenever a Demo is ended (due to leaving the lobby, or the server changing maps, etc...)
    * that MC should trigger a SDCM send operation with an additional FINAL identifier attached to that message.
- Whenever a Demo is begun (due to joining a lobby or new map, etc...)
    * that MC should trigger a SLSM send operation, with an additional DEMO_START identifier attached to that message.

{% include img-full.html path="assets/png/brick-wall-protocol.png" alt="Generic example of the Brick Wall protocol" %}
> Fig 1: A generic sample of a transaction between an MC and the MB during a game involving a late send/chunk skip from the MC, which results in a connection drop.

### The CDCACR Protocol (The Interceptor Protocol)

The Convict Detection and Consensus Agreement Coordination Response protocol is required for all MCs to implement and enact if they wish to remain a part of the LCP. Further, failing to action the CDCACR protocol in a lobby with other MCs that _do_ enact the protocol will result in a trust factor degradation and potentially being dropped from the MDSCP connection.

To implement CDCACR, an MC must do the following:
- Every time a MAC convicted player joins the lobby:
  * A reasonable attempt-scaling timer must be implemented to prevent popcorn abuse.
  * The currently recording demo should be split (stopped and then restarted)
  * a SDCM send operation should be triggered
  * the MB corroborates this information by querying the server scoreboard
    * if information correct, MB awaits messages from other MAC users
      * if an MC doesn't send a SDCM to the MB within a reasonable timeframe, that MC's trust factor is reduced.
    * Else the trust factor of the sending user is reduced for sending incorrect data
  * All MCs who reported the presence of the convicted player are instructed to vote-kick the convicted player as soon as available.
    * When the convicted player fully joins the game and gets assigned a team:
      * The MCs who cannot kick them because they are on the other team trigger a Standard Lobby Scoreboard Message send (SLSM) operation
      * The players who can kick them will attempt to call a vote, then split their demo and trigger a SDCM send operation
      * Upon receiving these Demo Chunks:
        * If the vote-kick is on the correct player, the MB will request all relevant MCs to 'F1' the vote.
        * If the vote is incorrect, but called by an MC that reported the convicted player, that MCs trust factor will be reduced and their MDSCP connection will be dropped.
    * (Note: All data received from demo splits in this phase will, as usual, implement the SAD protocol)
  * Upon the resolution of the vote-kick, regardless of success or failure, all engaged MCs shall trigger a SLSC send operation
    * The MB may respond to any number of engaged MCs with a request for an SDCM operation
      * if ignored, the given MC will suffer a trust factor reduction and be dropped from the LCP.

<!-- DIAGRAM TBD -->

**Notes**

Avoid performing Demo splits and SDCM send operations if possible, as this has a large performance impact when restarting the Demo.

'Popcorning' is the term used to refer to bots/cheaters that will continuously disconnect and reconnect to a server. 

# Appendix

## Standard Message Formats
### Standard Demo Chunk Message Struct:
- the `gzip` compressed demo chunk, 
- the current logged in steam user (steamID64),
- the current system time, 
- the current game tick,
- the SHA-2 hash of the concatenated message of:
   * entire past demo up until before this chunk (block aligned, PKCS#7)
   * the current logged in steam user (steamID64),
   * the current system time, 
   * the current game tick,
- the message shall be signed using the users PKA signing key
- the message shall be communicated securely using TLS/HTTPS only, with a valid user certificate.

### Standard Lobby Scoreboard Message Struct:
- The list of all players in the lobby, excluding their steamData, tags, customData, internalData and convicted status.