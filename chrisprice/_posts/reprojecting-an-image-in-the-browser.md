#Reprojecting an Image in the Browser

Recently whilst browsing Hacker News, I stumbled across an interesting app called PlaceIt by Breezi, which lets you "generate product screenshots in realistic environments". You just drag your screenshot onto one of the photos from their gallery and the app spits the gallery photo with your screenshot visible on the device. I'm sure you'll agree the end result looks very nice -

![PlaceIt example](placeit-example.png)

Intrieged, I fired up the dev tools to get a peek at what was going on. Expecting to see some hardcore canvas action, I was disappointed to find that it was all being done server-side. "But this is 2013" I hear you cry, "surely it can be done client-side". I concur - let's see if we can't fix that.

##The Basics

An educated guess would suggest to me that the final image is made up of 3 layers -
* A background image contains the bulk of the picture, with a blank spot where the screenshot will go. 
* A reprojected screenshot image onto the device in the picture's screen's co-ordinate space (i.e. bent and squashed to fit on the screen) 
* And that just leaves any glossy reflections and occlusions (e.g. fingers) for the final overlay layer.

_There's possibility also a separate mask layer and the background may be part of the overlay, but let's go with what's above for now._

I'm going to assume everyone can think up a few different techniques for compositing the layers in a browser and skip to the juicy stuff of reprojecting the screenshot.

##2D Transforms and Triangles

I think of a 2D transform, the sort of thing that Context2D.setTransform() or matrix(...) accepts, as something that can transform any triangle in the source image into any triangle in the destination image. That makes it ideal if the shape of your source image is triangular you want it to stay triangular in the destination image -

TRIANGLE TRANSFORM (with points highlighted)

Or even rectangular, so long as the source and destination shape is a parallelogram (opposite sides are parallel) -

PARALLELOGRAM TRANSFORM

But not so great if your destination shape isn't a parallelogram -

MISALIGNED PROJECTION

As you can see the bottom right of the screenshot is clearly in the wrong position. Now it would be possible to cut the source image up into a grid of smaller triangles and project each one with an appropriate transform to get an approximation of the real transform. Obviously, two triangles would leave a very visible join down the center of the image but as you increase the number of triangles you should get better and better results. 

However, wouldn't it be nice if we could do it in just one operation...

##3D Transforms and Quadrangles (Wonky Rectangles)



##Solving

Unfortunately it's time for a bit of maths, but I'll try and keep it brief! We're trying to solve a linear system of simulataneous equations, so we can bring to bear some basic matrix techniques to make our lives easier. Specifically we're going to use [Gaussian elimination](http://en.wikipedia.org/wiki/Gaussian_elimination), followed by a bit of back-substitution.

Here's the Gaussian elimination example from the Wikipedia article, this is a 3x3 matrix but it's generalisable upto NxN -

The simultaneous equations

![Linear system of simultaneous equations](http://upload.wikimedia.org/math/a/f/0/af049b44d89484bcc6114dde940d4edc.png)

Turn into the augmented matrix

![Augmented matrix](http://upload.wikimedia.org/math/a/e/c/aec68ce94e1b6e1ff6ceec8b101fb1a8.png)

Which can be converted into row echelon form

![Row echelon form](http://upload.wikimedia.org/math/b/5/c/b5c5821f745a153ecb193c7f329eaad5.png)

And then into reduced row echelon form

![Reduced row echelon form](http://upload.wikimedia.org/math/f/2/9/f2981fd8dffb705698e90dbcfcea25d5.png)

The results can then be read off as x = 2, y = 3 and z = -1.

So how do we implement this in code? Well there's always the pseudo-code on the Wikipedia article if you're feeling adventurous, but I opted for an existing matrix library called [Sylvester](http://sylvester.jcoglan.com/). I checked out the API docs and found the [toRightTriangular method](http://sylvester.jcoglan.com/api/matrix.html#torighttriangular). Adapting the code given there to match the example above -

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

We're close, all we need to do now is generalise the back-substitution step (the bit that gives us the reduced row echelon form) to support NxN matrices. Again if you're feeling adventurous the pattern should be obvious so feel free to get your algorithm on, I instead took inspiration from the [matrix solver by Stephen R. Schmitt](http://mysite.verizon.net/res148h4j/javascript/script_gauss_elimination3.html) and implemented a generalised back-substitution like so -

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

So now we can combine our solver with the matrix from the previous section to create our reprojection matrix, but how do we actually go about using it?

_As is always the case once you know the right terms, finding the [generalised solution on Google](http://www.google.com/search?q=rectangle+to+quadrilateral+mapping) is trivial. If you want to implement this yourself, you may find it easier to use it rather than using the above technique._

##Reprojecting using CSS3 Transforms

As we discovered earlier, a 2D transform matrix just isn't going to cut it, so we know we're going to need to use the [matrix3d transform function](https://developer.mozilla.org/en-US/docs/CSS/transform-function#matrix3d()). In order to make the reprojection matrix fit the format expected by the browser we'll pre-process the matrix in 2 ways -
* Transpose the matrix (swap each element with the element on the opposite side of the diagonal)
* Add a no-op transform for the 3rd dimension (z)

That gives us something like -

```
  var t = solve(...);
  [
    [ t[0], t[3], 0, t[6]],
    [ t[1], t[4], 0, t[7]],
    [    0,    0, 1,    0],
    [ t[2], t[5], 0,    1]
  ]
```

Which we can apply directly to an element like so -

```
  var matrix = new WebKitCSSMatrix();
  matrix.m11 = t[0], matrix.m12 = t[3], matrix.m13 =    0, matrix.m14 = t[6];
  matrix.m21 = t[1], matrix.m22 = t[4], matrix.m23 =    0, matrix.m24 = t[7];
  matrix.m31 =    0, matrix.m32 =    0, matrix.m33 =    1, matrix.m34 =    0;
  matrix.m41 = t[2], matrix.m42 = t[5], matrix.m43 =    0, matrix.m44 =    1;
  element.style.webkitTransform = matrix;
```

_I'm using a WebKit only class (and prefix) for brevity, you'll need to fall back to string concatenation for other browsers e.g. `"matrix3d(" + t[0].toFixed(x) + "," + ... + ")"`._


Unfortunately that on it's own doesn't quite give the expected result -

#BROKEN

The problem is that the browser is taking the origin of the image to be at the center of the image but when we computed the matrix we assumed the origin was in the top left of the image. Luckily, we can fix that with a little bit of `element.style.webkitTransformOrigin = "0 0"` to get the result we're after -

#FIXED

##Reprojecting using WebGL/three.js
