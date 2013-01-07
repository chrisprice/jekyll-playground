#Reprojecting an Image in the Browser

Recently whilst browsing Hacker News, I stumbled across an interesting app called PlaceIt by Breezi, which lets you "generate product screenshots in realistic environments". You just drag your screenshot onto one of the photos from their gallery and the app spits the gallery photo with your screenshot visible on the device. I'm sure you'll agree the end result looks very nice -

![PlaceIt example](placeit-example.png)

Intrieged, I fired up the dev tools to get a peek at what was going on. Expecting to see some hardcore canvas action, I was disappointed to find that it was all being done server-side. "But this is 2013" I hear you gasp in horror, "surely it can be done client-side". I concur - let's see if we can't fix that.

##The Basics

An educated guess would suggest to me that the final image is made up of 3 layers -
* A background image contains the bulk of the picture, with a blank spot where the screenshot will go. 
* A reprojected screenshot image onto the device in the picture's screen's co-ordinate space (i.e. bent and squashed to fit on the screen) 
* And that just leaves any glossy reflections or occlusions for the final overlay layer which is again composited on top.

_There's possibility also a separate mask layer, but for the sake of argument let's assume that's folded into the overlay._

I'm going to assume everyone could think up a few different techniques for compositing the layers in a browser and skip to the juicy stuff of reprojecting the screenshot.

##2D Transforms and Triangles

##3D Transforms and Quadrangles (Wonky Rectangles)

##Solving

Unfortunately it's time for a bit of maths, but I'll try and keep it brief! We're trying to solve a linear system of simulataneous equations, so we can bring to bear some basic matrix techniques to make our lives easier. Specifically we're going to use [Gaussian elimination](http://en.wikipedia.org/wiki/Gaussian_elimination), followed by a bit of back-substitution.

Here's the Gaussian elimination example from the Wikipedia article, here it's used on a 3x3 matrix but it's generalisable upto NxN -

The simultaneous equations

![Linear system of simultaneous equations](http://upload.wikimedia.org/math/a/f/0/af049b44d89484bcc6114dde940d4edc.png)

Turn into the augmented matrix

![Augmented matrix](http://upload.wikimedia.org/math/a/e/c/aec68ce94e1b6e1ff6ceec8b101fb1a8.png)

Which can be converted into row echelon form

![Row echelon form](http://upload.wikimedia.org/math/b/5/c/b5c5821f745a153ecb193c7f329eaad5.png)

And then into reduced row echelon form

![Reduced row echelon form](http://upload.wikimedia.org/math/f/2/9/f2981fd8dffb705698e90dbcfcea25d5.png)

The results can then be read off as x = 2, y = 3 and z = -1.

So how do we implement this in code? Well there's always the pseudo-code on the Wikipedia article if you're feeling adventurous, and for the rest of us there are existing matrix libraries to do it for us. I opted for [Sylvester](http://sylvester.jcoglan.com/), checked out the API docs and found the [toRightTriangular method](http://sylvester.jcoglan.com/api/matrix.html#torighttriangular). Adapting the code given there to match the example above -

```
  var equations = $M([
    [ 2,  1, -1,   8],
    [-3, -1,  2, -11],
    [-2,  1,  2,  -3]
  ]);

  var eqns = equations.toRightTriangular();

  var sol_z = eqns.e(3,4) / eqns.e(3,3);
  var sol_y = (eqns.e(2,4) - eqns.e(2,3)*sol_z) / eqns.e(2,2);
  var sol_x = (eqns.e(1,4) - eqns.e(1,3)*sol_z - eqns.e(1,2)*sol_y) / eqns.e(1,1);
  
  console.log(sol_x, sol_y, sol_z); 
  // 2, 3, -1
```

We're close, all we need to do now is generalise the back-substitution step (the bit that gives us the reduced row echelon form) to support NxN matrices. Again for the adventurous feel free to get your algorithm on while the rest of us hit Google up. I've taken inspiration from the [matrix solver by Stephen R. Schmitt](http://mysite.verizon.net/res148h4j/javascript/script_gauss_elimination3.html) and implemented a generalised back-substitution like so -

```
  var result = [], rowCount = eqns.rows();
  for (var i = rowCount - 1; i >= 0; i--) {
    var row = eqns.elements[i];
    for (var sum = 0, j = i + 1; j < rowCount; j++) {
      sum += row[j] * result[j];
    }
    result[i] = (row[rowCount] - sum) / row[i];
  }

  console.log(result); 
  // [2, 3, -1]
```

##Reprojecting using CSS3 Transforms

##Reprojecting using WebGL/three.js
