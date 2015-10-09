---
layout: default
title: The Deeper Unreal Engine 4 Gameplay Framework Overview
---

This file describes core Unreal Engine 4 game framework components and their relationships.

## UObject

Abstract object, managed by garbage collector. Created via NewObject, deleted automatically after GC detects that it is no longer reachable from GC entry points directly or via references from other UObjects

## UWorld : UObject

3D space with coordinates where things can live. UWorld can either be autonomous (working on its own, in standalone game or on server) or be a slave connected via network to authority world.

## AActor : UObject

Thing that lives inside UWorld. Actors can have ownership relations, which matters when player is given control upon actor. Player can only invoke RPC commands on actors it owns (either directly or indirectly).

## UActorComponent : UObject

Composable AActor part. Is used to split actor functionality into reusable components. Some ActorComponents have world representation (meshes, collisions, particles), some are just abstract concepts like MovementComponent.

## AGameMode : AActor

Autonomous-world thing (not created on networked clients) whose primary responsibility is world-wide gameplay rules handling like match state, player spawning, etc. It is the first Actor that BeginPlay is called.

## AGameState : AActor


Replicated thing whose primary responsibility is storing world-wide game progress. Primary GameState is created in autonomous world, networked clients receive GameState via replication.

## AController : AActor

Something that can produce input to influence the world. Either player or AI.

## APlayerController : AController

Replicated thing that represents a player in autonomous world. In networked games, PlayerController is attached to network connection and server allows connected client to pass commands to its PlayerController. PlayerController is only replicated to client it is attached. Is destroyed if player gets disconnected.

## AHUD : AActor

Client-only thing whose primary goal is to provide events (BeginPlay/EndPlay) for game GUI. HUD is created for a single PlayerController and has a reference to it.

## APlayerState : AActor

Replicated thing that stores per-player game progress (like score, kills/deaths count). When new player connects to the server, it is either given a new PlayerState or existing one can be given (if game supports reconnects without losing game progress). It is the longest-living object related to specific player on server.

## ULocalPlayer : UObject

Client-only thing, connects devices (keyboard, mouse, gamepad, etc) to PlayerController. In split-screen games, there can be multiple LocalPlayers (for example, one per gamepad).

## FViewportClient

Virtual screen assigned to ULocalPlayer. In split-screen games, each LocalPlayer is assigned a separate viewport. There also can be games where LocalPlayers share a single viewport (like Mortal Kombat). And there can be games where a single viewport is passed from one player to another (like Heroes of Might and Magic HotSeat mode).

## APawn : AActor

Replicated thing that can be owned (possessed) by a Controller. Ownership is transitive across all Actors (if Pawn is a weapon owner and PlayerController is a pawn owner, then this PlayerController is allowed to call RPC functions on this weapon, firing it for example).

## ACharacter : APawn

Replicated thing with humanoid-specific functionality, mostly movement. Can walk, jump, swim, crouch.

## AGameSession : AActor

Game-specific object defining a "session" scope. Can live longer than a World in games where one session implies several world games. Can know what players are allowed to belong to it.

## AGameInstance : AActor

A context that carries state across worlds. Has a concept of "current world". For example, you first start game in menu world, then transfer to loading screen world and then to game world. All this happens inside a single GameInstance that switches its current world.

## UEngine : UObject

A thing that ticks GameInstances, Worlds and other application-wide helper stuff. Theoretically, there could be several Engines in one UE process, each ticking its own games, but currently it is impossible due to heavy usage of static GEngine variable.

## How standalone game starts (both single-player and split-screen)

 1. You start game executable
 2. UEngine is created
 3. UEngine looks up what GameInstance your game wants and creates one
 4. GameInstance creates startup world with all actors placed in it
 5. LocalPlayer(s) are created
 6. PlayerController(s) are created and linked to LocalPlayers
 7. GameMode is created
 8. GameSession is created
 9. GameState is created
 10. GameMode begins play
 11. Depending on game rules, PlayerController(s) can be given Pawn(s)
 12. UEngine ticks the whole thing to animate game

The most challenging aspect of multi-LocalPlayer games (splitscreen, hot seat, etc) is that you must always be very careful not to mix up LocalPlayers and their PlayerControllers, especially given the fact that they all live within a single World.

## How dedicated server starts

 1. You start game executable
 2. UEngine is created
 3. UEngine looks up what GameInstance your game wants and creates one
 4. GameInstance creates startup world with all actors placed in it
 5. GameMode is created
 6. GameSession is created
 7. GameState is created
 8. GameMode begins play
 9. NetDriver is created by World and starts listening for client connections

## How client joins networked game

 1. Client connects to server and negotiates whether it is allowed to join and what map it is supposed to load
 2. Client loads same map as currently loaded on server
 3. On server, PlayerController is created and linked to client network connection
 4. PlayerState is either created or existing one looked up and assigned to PlayerController
 5. Client waits until PlayerController is replicated to it and links it to LocalPlayer
 6. Info about other replicated actors is transmitted to client so it can create their "copies" in its world and animate them repeating what they do on server
 6. Client confirms server that it is ready to play
 7. Depending on game rules, Pawn can be assigned to PlayerController on server and then replicated to client

## How client leaves networked game

 1. Client either shutdowns or loads another world
 2. Previous world is destroyed on client, what leads to network connection close
 4. On server, PlayerController unpossesses Pawn, if it had any
 3. Server destroys PlayerController
 4. Server either destroys PlayerState or marks as "inactive" to reassociate it with same player if they connect again
 5. Other clients are notified that PlayerState is no longer replicated to them, so they delete its replicated copies.


## How listen server starts

Same procedure as in standalone game is performed, but additionally after GameMode began play, world starts listening to network (via NetDriver). When clients connect, listen server treats them the same way as dedicated server does.

## Networking gone crazy

UE supports split-screen (better to say, multi-LocalController) game acting as either listen server or networked client. When acting as client, it tells server "hey, there are 2-3-4-N of us here, please give us enough PlayerControllers". It is client responsibility to determine how to match PlayerControllers to LocalPlayers. Such setup allows to reduce network traffic because server replicates world actors only once regardless of the number of LocalPlayers viewing the world from one client.

## How PIE works

Depending on "dedicated server" checkbox and number of clients, either standalone game (1 player, dedicated=off) or listen server + clients (>1 player, dedicated=off) or dedicated server + clients (any number of players, dedicated=on) are created. All are ticked by single UEngine (remember, it is able to tick multiple worlds).

## Multisession server

It is possible to run a server that handles multiple worlds at once (though by a single thread). Each world will have a separate netdriver and listen on different ports. It is even theoretically possible to serve multiple sessions on a single port, though it will definitely require a custom NetDriver implementation, mainly to figure out how to distribute incoming connections between worlds.

As was already said, it isn't currently possible to run multiple UEngines (thus, utilizing multiple cores) in a single UE process due to static variables. Let's hope UE 5 will have this feature.

## Detecting world mode

`World::GetNetMode()` can return:

 * `NM_Standalone` - autonomous world, not listening to network. This is what single-player games have.
 * `NM_ListenServer` - autonomous world with LocalPlayers, listening to network. This is what happens when you host a networked game and other clients connect to you
 * `NM_DedicatedServer` - autonomous world without LocalPlayers, listening to server.
 * `NM_Client` - slave world, connected via network to listen/dedicated server

Technically speaking, world netmode isn't fixed during world lifetime. Nothing prevents autonomous world from starting and stopping accepting networked connections, what would switch its mode between NM_Standalone <-> NM_ListenServer. Also, nothing prevents LocalPlayers from entering and quitting existing worlds, what would switch world mode between NM_ListenServer and NM_Dedicated. Though there is no guarantee that all UE parts are prepared for such changes, not to say about game itself.

There are two common checks that are done for net mode:

 * `NetMode != NM_Client` allows to check if this is an autonomous world
 * `NetMode != NM_DedicatedServer` allows to check if there are local players, so it makes sense to spawn audio/visual effects. Though more correct way is to check LocalPlayers existence directly.

# Detecting actor network role

`Actor::Role` can have different values:

 * `ROLE_Authority` - actor is autonomous, either living in an autonomous world or created locally in slave world
 * `ROLE_SimulatedProxy` - actor replica on networked client. Always has paired authority actor living in autonomous world.
 * `ROLE_AutonomousProxy` - actor replica on networked client that has control over this actor. Always has paired authority actor living in autonomous world. Only PlayerControllers and Pawns ever get this role currently.

There most common check for actor role is `Role == ROLE_Authority` that allows to check if it is a primary actor object or just its replica.

## How server travel works (non-seamless)

TODO

## How server travel works (seamless)

TODO
