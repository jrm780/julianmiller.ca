+++
date = "2017-06-27T00:00:00+06:00"
title = "Paddles in a Networked Game: A Real-Time Interactive Simulation"
categories = ["Software", "Education"]
tags = ["Ping", "Programming", "Networking", "Game"]
+++
The contents of this article were originally written for a research paper several years ago. It is a general overview and implementation of common network programming techniques still heavily used in online video games.

# 1. Introduction

In this report, the fundamentals of a real-time interactive simulation consisting of multiple clients connected over a network are examined and implemented. Paddles in a Networked Game (Ping) is a tennis-like computer game in which two players each control a paddle at either end of the screen. A ball bounces between the paddles, and players score by making the ball pass beyond the opposing player's paddle. Due to the real-time interactivity of Ping, propagation delay over the network can become troublesome. Therefore, techniques including client-side interpolation and prediction are used to provide an acceptable user experience.
![PING](blog/ping/ping.PNG)

## 2. Background

Several techniques have been developed to handle the issue of network latency. Generally, the user is unaware of their presence, but, without these techniques, real-time interactive applications would be rendered unusable over standard IP [<sup>1</sup>](/blog/ping#references).   The most common techniques include data compression, interpolation, prediction, dead-reckoning, and lag compensation. Ping focuses on client-side input prediction and interpolation with respect to client-server architecture.

### 2.1 Client-Server Architecture

Many games use a client-server architecture in which a central game server allows players to connect to it through game clients. The server is usually authoritative, meaning it has final say with regard to the score of players and where entities are positioned [<sup>3</sup>](/blog/ping#references). Naturally, the client and server must communicate in a way such that their states are synchronized. One method of synchronization could be for the clients and server to run their simulations with identical code and have the client periodically inform the server of its state. For example, `client A` tells the server its paddle is currently at position `(x,y)`, and the server incorporates this information into its own state, subsequently informing the other clients about the position of `client A.`
![Client-server](blog/ping/ping-client-server.PNG)

However, this introduces the ability to cheat, as `client A` could give the server a "fake" position. In fact, a player could effectively teleport from one position to another by giving the server two very different positions, one after the other.

In order to prevent cheating and to ensure the clients aren't performing game-breaking maneuvers (e.g., teleporting), a better approach is for the clients to inform the server only about player input [<sup>2</sup>](/blog/ping#References). For example, `player 1` presses a button on her controller, and the game client sends a message to the server informing it of the button-press. The server can then process the input, update its state accordingly, and send the new state to all of the clients. The players may still be able to send "fake" input, but the input will always result in a valid maneuver as defined by the server. However, this creates another problem; because the input message must travel to the server and the new server state must travel back to the client, there is a noticeable delay between when `player 1` presses a button and when `player 1` actually sees the effect of pressing the button.

### 2.2 Client-Side Prediction

Assume a player has pressed a button to move left, and there is a network round trip time of 100 milliseconds. The `move left` message is sent to the server, and the resulting state is received by the client 100 milliseconds later. This means the player would see his own change in movement with a 100 millisecond delay. This phenomenon feels unnatural, and becomes worse with higher latencies.
![Delay](blog/ping/ping-delay.PNG)

In order to overcome this and provide immediate feedback for user input, the game client must somehow predict the player's movement and immediately present the resulting position to the user before it has received the updated server state. In order to do this, the client simply runs the exact same code as the server, so the client can determine the effect of the received input. Note that Ping only predicts local movement, as it is impossible to predict what other players might do. This indeed implies that other clients are actually viewed 100 milliseconds in the past.
![Prediction](blog/ping/ping-prediction.PNG)

After the client prediction has taken place, the client will still eventually receive the updated server state. Because the server is authoritative, if the predicted state somehow does not match the server state, the server state must be accepted as the final state. The effect of these errors can be minimized by gradually correcting the difference in entity positions in a smooth manner. Another server reconciliation method is to store the input on the client, and reapply it to the new state, after it is received from the server. In Ping, after a player scores a point, the positions of the ball and paddles are reset, which removes any discrepancies between states. Because scoring occurs frequently, this is an effective method of state reconciliation.

### 2.3 Interpolation

Sending the server state to clients every time it is updated would produce a lot of unnecessary network traffic. This would become especially problematic on low-bandwidth connections. Instead, the server sends snapshots of its state to the clients at a much lower rate. Ping updates its simulation 60 times per second, but the server sends its state to clients at a rate of only 20 times per second. Because the clients receive server state updates at a low rate, the game appears choppy. Interpolation combats this issue by computing the positions of entities between two received states [<sup>7</sup>](/blog/ping#References).

Ping stores the two most recently received state snapshots from the server. When the client receives a new snapshot, the oldest is discarded. Now, instead of rendering the most recent state to the screen, the client renders a state interpolated appropriately between the two stored states. This provides a much smoother animation, as it appears as though the server states are being received at the full simulation speed of 60 times per second.

```C++
void GameClient::update()
{
    const float snapshotDelta = snapshot.time - prevSnapshot.time;
    const float alpha = (localTime - prevSnapshot.time - 0.05f) / snapshotDelta;
    player.position = alpha*snapshot.playerPosition +
                      (1-alpha)*prevSnapshot.playerPosition;
}
```

## 3. Implementation

Ping is written in C++ and makes use of the Simple and Fast Multimedia Library (SFML) [<sup>5</sup>](/blog/ping#References). SFML was chosen because it is a cross-platform library that allows for higher-level object-oriented OpenGL rendering and socket programming. Ping uses a client-server architecture, where players run game clients on their machines which connect to a server hosted on another machine. An abstract Game class is used as a controller that handles the initialization of resources, socket connections, and the main update loop. The update loop occurs at a fixed 60Hz to ensure determinism across the server and clients.
![PING](blog/ping/ping-game-diagram.PNG)

Like many of the other Ping classes, Game implements an Updateable interface, which gives it an update() method; this method is executed at every iteration of the main loop. Two Paddle objects and a Ball object are contained in a Stage object (Figure 2), which is contained in the Game object. Because client-side prediction relies on the client and server running identical simulations, both the GameClient and GameServer inherit the Game class, giving them similar attributes.
![PING](blog/ping/ping-stage-diagram.PNG)

### 3.1 The Server

Upon initialization, the GameServer waits for incoming client TCP connections. TCP is used, because Ping depends on reliable and in-order packet transferring. SFML also disables Nagle's Algorithm, which can otherwise be problematic when immediate packet sending is required. Furthermore, the development of a lightweight and reliable UDP "connection" is outside the scope of this project. When a client connects, the server responds with a ClientConnected message and assigns that client a playerIndex, so the client is aware of which paddle it is controlling.  After the second client connects and has been assigned a playerIndex, the GameServer sends a GameStarted message to the clients and enters the main update loop.

The Ball begins at the centre of the Stage and is released in a pseudo-random direction after a three-second countdown. A BallReleased message is sent to the clients when this occurs. Each subsequent iteration simply updates the GameServer's Stage, which simulates the physics of the Ball, assuming a velocity of constant magnitude and perfectly-elastic collisions with the boundaries of the Stage. Prior to updating the Stage, however, the GameServer applies any client input to the paddles, if a PlayerMoved message has been received. When a player has scored, the clients are notified by a PlayerScored message, the Stage is reset, and the three-second countdown begins once again.

### 3.2 The Client

When the GameClient is initialized, it attempts to establish a TCP connection with the supplied server IP address. After a connection has been established, a ClientConnected message is expected to be sent from the server to the GameClient. The ClientConnected message informs the GameClient of which paddle it is controlling. After both GameClients have connected to the server, a GameStarted message is sent by the server to the clients. At this point, the GameClient opens the graphical display (Figure 3), and the user is presented with a window which displays the three-second countdown.

Once the Ball is released, the player is able to move her paddle left or right using the keyboard. Each key-press and key-release sends a PlayerMoved message to the server. Additionally, the GameClient uses the same Stage update algorithm as the GameServer, which effectively predicts the resulting paddle position and gives the user immediate feedback of his input. In order to view the effect of prediction, it may be toggled on and off by the user.

### 3.3 Synchronization

When a player has scored, the server notifies the clients of this event through a PlayerScored message, and the server and clients reset their Stages. This resets all entity positions such that the server and clients all match each other. Additionally, the GameServer sends GameSync messages to the clients at 20Hz. Each GameSync message contains a snapshot of the server–including scores, paddle, and ball position at the time the message was sent. The GameClient then sets its entity positions to match the server state.

The GameSync rate is made much lower than that of the 60Hz main update loop to minimize the amount of network traffic required by the simulation. In order to provide a quality experience at the user-end, client-side interpolation is employed to offer a smooth transition between the received states. This interpolation may be toggled on and off by the user, to gain a better understanding of its effect. Table 1 outlines all of the messages passed between the server and clients in order to keep them synchronized.

  Message Type    |  Included Data           | Description
 ---------------- | ------------------------ | ------------
  ClientConnected | int: the player index    |  Sent from server to client when a client connects
  GameStarted     |                          |  Sent from server to client when the game begins
  PlayerMoveLeft  | bool: start or stop      |  Sent client to server when player has started/stopped moving left
  PlayerMoveRight | bool: start or stop      |  Sent client to server when player has started/stopped moving right
  PlayerScored    | int: the player index    |  Sent from the server to client when a player has scored
  BallReleased    | float: the ball angle    |  Sent from the server to the client when the ball is set in motion
  GameSync        | snapshot of server state |  Sent from server to client 20 times per second

## 4. Results

The implementation of Ping was tested on Microsoft Windows over a wireless local-area-network. The server was hosted on a desktop with an Intel Core 2 Quad Q8800 processor and 4GB DDR3 memory. One of the game clients was also run on the server. The other client was run on a laptop with an Intel Core i5 processor and 8GB DDR3 memory. The laptop also made use of SoftPerfect Connection Emulator [<sup>6</sup>](/blog/ping#References), which simulates a wide area network connection and allows for variable conditions, such as bitrate, latency, and packet loss. During average play, Ping consumed 28.1Kbps downstream and 11.5Kbps upstream. Therefore Ping was completely playable and ran smoothly on a simulated 48.0Kbps modem. Ideally, this could be further optimized. Additionally, latency was required to exceed 150 milliseconds before it became noticeable.

## 5. Conclusion

There are many techniques developers can implement to cope with the effects of network latency in real-time interactive applications. While Ping does not make use of all of them, a combination of efficient message passing, prediction, and interpolation provide an acceptable user experience. By sending fewer game synchronization messages, network traffic is reduced and congestion is avoided. Without client-side interpolation, this low synchronization rate would not be possible, as it would produce poor animation quality on the client-side. Input prediction also provides immediate feedback for user input, eliminating the potential effects of input lag.

The source code for Ping is available on GitHub: https://github.com/jrm780/Ping

## References

1. Anthony Steed. 2011. Introduction to networked graphics. In SIGGRAPH Asia 2011 Courses (SA '11). ACM, New York, NY, USA, , Article 12, 159 pages. DOI=`10.1145/2077434.2077445` http://doi.acm.org/10.1145/2077434.2077445.
2. G. Fiedler. Networking for Game Programmers, http://gafferongames.com.
3. G. Gambetta. Fast-paced multiplayer, http://www.gabrielgambetta.com/?p=11.
4. Marescaux J, et al. Transatlantic Robot-Assisted Telesurgery. Nature 2001;413:379–380.
5. Simple and Fast Multimedia Library, http://www.sfml-dev.org/.
6. SoftPerfect WAN Connection Emulator for Windows, http://www.softperfect.com/.
7. Source Multiplayer Networking, https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking.
