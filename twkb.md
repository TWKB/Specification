#TWKB

version 0.21

Specification for "Tiny WKB", TWKB
	
##Abstract 

TWKB is meant to be a multi purpose slimmed format for transporting vector geometry data.
	
##The structure

###General rules
The first point in a TWKB-geometry is represented by it's full coordinates, the rest is delta-values relative to the point before.
How the deltavalues are stored as varInts<br><br>
Datatypes used is of fixed length or VarInt. VarInt is a vay of encoding variable length Integers described here:
https://developers.google.com/protocol-buffers/docs/encoding#varints

###The main header

1 byte<br>
the first byte of the TWKB-geometry only apears once, no matter what type it is.

That byte is used like this:

bit 1	**Is there an ID?**
* set: The TWKB or subgeometries have ID, see the specific types
* unset: No ID and no space for ID

bit 2	**Is size-information included?**
* set: Yes, directly after this byte comes a varInt holding the size of the rest of the geometry
* unset: No size-info and no space for size-info

bit 3	**Is bounding boxes included?**
* set: Yes, after the size information is the bounding box
* unset: No no bounding box and no space for a bounding box

bit 5-8 **precision:** tells how many decimals to use in coordinates see section "Storage of coordinates"
	
###Optional extra information

####Sizes
1 varInt <br> 
If the size information bit is set a varInt with size info comes next. The size given is the rest of the size after this size information.<br>
That means that when the application has read this size information, by adding this size value to the cursor it is standing in front of the next geoemtry.<br>
In that way it is very fast to just scan through the geometries to get the bounding boxes or to distribute geometries to different threads to be read.

###The type

* 1 byte holding geometry **type** and **number of dimensions**

this byte with type information will apear after the initial byte in every twkb-geometry, and for each geometry in a geometry collection

The type-byte is used like this:

bit 1-5 gives 31 type positions, we use a few of them:

* 1	Point
* 2	Linestring
* 3	Polygon
* 4	MultiPoint
* 5	MultiLinestring
* 6	MultiPolygon
* 7	GeometryCollection
* 21	MultiPoint with id on each point
* 22	MultiLinestring with id on each linestring
* 23	MultiPolygon with id on each polygon
* 24	GeometryCollection with id on each collection
* 25	TopoLinestring
* 26	TopoPolygon

bit 6-8:  number of dimensions (ndims)

#### Bounding box
4 varInt<br> 
If the bounding box bit in the first byte is set a bounding box comes next.
A bounding box is represented with varInts like:<br>
xMin, deltax, ymin, deltay, .......
This way a 2d box can be read without knowing the number of dimmensions (and then juming to next geoemtry. If the geoemtries is to be read all dimmensions of the bbox must of course be read.)

###Description type by type

#### Type 1, **Point**
* varInt **ID**, optional, only used if very first bit of TWKB is set
* the point coordinates (as deltavalue if in a MultiPoint or Geometry Collection)
	
#### Type 2, Linestring
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below

#### Type 3, Polygon
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **nrings** holding number of rings (first ring is boundary, the others are holes)

For each ring {<br>
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

#### Type 4, MultiPoint (with one id for all)
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **npoints** holding number of points
* a Point Array see section "Delta value array rules" below

#### Type 5, MultiLineString (with one id for all)
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **nlinestrings**  holding number of linestrings

For each linestring {<br>
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

#### Type 6, MultiPolygon (with one id for all)
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **npolygons** holding number of polygons

For each polygon {<br>
* varInt **nrings** holding number of rings (first ring is boundary, the rest is holes)

For each ring {<br>
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
  }<br>
}	

#### Type 7, GeometryCollection 
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **ngeometries** holding number of geometries

For each geometry {<br>
* varInt describing type and ndim  of subgeometry<br>
* a geometry of the specified type without ID<br>
}

#### Type 21, MultiPoint (with individual id)

* varInt **npoints** holding number of points

For each point {<br>
* Point of type 1
}


#### Type 22, MultiLineString (with individual id)

* varInt **nlinestrings** holding number of linestrings

For each linestring {<br>
* Linestring of type 2
}

#### Type 23, MultiPolygon (with individual id)

* varInt **npolygons** holding number of polygons

For each polygon {<br>
* Polygon of type 3
}

#### Type 24, MultiGometryCollection (with individual id)

* varInt **collections** holding number of collections

For each collection [MultiPoints, MultiLinestrings, MultiPolygons or GeometryCollections) {<br>
* MultiPoint of type 4
or
* MultiLinestrings of type 5
or 
* MultiPolygon of type 6
or
* GeometryCollection of type 7
}

#### Type 25	topo linestring
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **ncomponents** holding number of components used to build the linestring
* array of id-values to linestrings or points (type 1,2 or members of type 7, 21 or 22) (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

#### Type 26	topo polygon
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **ncomponents** holding number of components used to build the polygon
* array of id-values to linestrings or points (type 1,2 or members of type 7, 21 or 22) (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

## Coordinates storage

All storage is **integers**. So what happens is that the value gets multiplied with 10^precision-value, and is then rounded to closest integer<br>When reading the twkb, the value should be divided with 10^precision-value

So if the precision value is 2, we multiply the value with 100 and rounds the result to closest integer when we create out twkb-geometry and do the reveresed operation when we read it.

## Delta value array rules

This is about how the coordinates are serialized. The problem to be solved is that we want to use as little space as possible to store our coordinates.<br>
To do that we cannot just use a fixed data type because than we have to use the biggest size possible necessary. Instead we have to find a way to change the storage size as the need changes. <br>
The delta value between two coordinates vary a lot.<br>
This is solved by encoding the values as varInt
The method is described here:
https://developers.google.com/protocol-buffers/docs/encoding#varints



