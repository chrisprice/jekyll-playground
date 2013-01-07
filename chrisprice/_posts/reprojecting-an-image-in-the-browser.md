#Reprojecting an Image in the Browser

Recently whilst browsing Hacker News, I stumbled across an interesting app called PlaceIt by Breezi, which lets you "generate product screenshots in realistic environments". You just drag your screenshot onto one of the photos from their gallery and the app spits the gallery photo with your screenshot visible on the device. I'm sure you'll agree the end result looks very nice -

![PlaceIt example](placeit-example.png)

Intrieged, I fired up the dev tools to get a peek at what was going on. Expecting to see some hardcore canvas action, I was disappointed to find that it was all being done server-side. "But this is 2013" I hear you gasp in horror, "surely it can be done client-side". I concur - let's see if we can't fix that.

##The Basics

An educated guess would suggest to me that the final image is made up of 3 layers, a background, a reprojected version of the screenshot and an overlay. The background contains the bulk of the picture, with a blank spot where the screenshot will go. The screenshot needs reprojecting onto the device in the picture's screen's co-ordinate space (i.e. bent and squashed to fit on the screen) and then compositing on top. And that just leaves any glossy reflections or occlusions for the final overlay layer which is again composited on top. There's possibility also a separate mask layer, but for the sake of argument let's assume that's folded into the overlay.

##2D Transforms and Triangles

##3D Transforms and Quadrangles (Wonky Rectangles)

##Solving

##Using with CSS3 Transforms

##Using in three.js
