---
layout: post
title: New Throwing Mechanic! Untitled Drone Puzzle Game
youtubeId: 0xasKAsCHio
---

Sharing another small updated on my as-yet untitled drone-based puzzle game.

I added a throwing mechanic to my untitled drone puzzle Unity game. This makes it a little easier for the player to insert objects such as power plugs and batteries into their slots from a distance, without having to awkwardly fly every object around.

The most difficult parts of implementing this mechanic were drawing the prospective flight path of the object, and getting the camerawork right. I think the camera still needs some work actually.

In order to draw an accurate throw path I had to do a discrete integration over time of the object's position to simulate Unity's discrete time step based physics. The path rendering is done using a standard Unity LineRenderer component. It seems pretty accurate! However, YMMV. The bigger the object and the less spherical it is, the less accurate this estimate will be. It also assumes no drag.                                                                                                    

{% include youtubePlayer.html id=page.youtubeId %}