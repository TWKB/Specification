# Triangulation Information in TWKB


	
## WHY?

When rendering in for instance openGL everything is one pixel point, on pixel wide line or triangles. A surface consists of one or many triangles.
Calculating triangles of a polygon is quite expensive, so TWKB have the possibility to store information about the triangles, so it can be done once, and rendered faster.
	
## HOW? 

When loading data to the gpu for rendering a polygon, one way is to load a list of coordinates and one list of indexes into the coordinates, telling how to build triangles

An example:

Coordinate list
1 1, 1 2, 2 2, 2 1

this is a square that can be divided in 2 triangles :
1 1, 1 2, 2 2
and 
1 1, 2 2, 2 1

in the coordinate list the coordinate have the following ordinates:
1 - 1 1
2 - 1 2
3 - 2 2
4 - 2 1

so we can describe the triangels from those ordinates like 

1 2 3, 1 3 4

So, what the gpu gets is a list of coordinates and those indexes into the coordinate list like
1,1,1,2,2,2,1
and
1,2,3,1,3,4

and the box can be rendered

## The TWKB-way of encoding this

If the twkb-triangulation flag is set in the first twkb byte, then after the polygon itself:

First a unsigned varint describing how many triplets we have (the number of triangles)

Then comes a list of unsigned varints (3 times as many as number of triangles)

To compress large polygons with maybe thousands or millions of points we use the delta value between first, second and third index 

If the implementation sorts the triplets in a smart way the size of the triangulation list can be reduced.

So, in our example above the triangle list is handled like this.

original:
123
134

So, first we write the initial triplet:
123

then we calculate delta values:

1 - 1 = 0

3 - 2 = 1

4 - 3 = 1

so, the second triplet is described like
011

The whole triangle list is now:
123011

For large polygons this is very efficient.

