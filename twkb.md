#TWKB

version 0.1

Specification for "Tiny WKB", TWKB
	
##Abstract 

TWKB is meant to be a multi purpose slimmed format for transporting vector geometry data.
	
##The structure

###General rules
The first point in a TWKB-geometry is represented by it's full coordinates, the rest is delta-values relative to the point before.
How the deltavalues are serialized is described in the section "Delta value array rules" below.<br><br>
Datatypes used is of fixed length or VarInt. VarInt is a vay of encoding variable length Integers described here:
https://developers.google.com/protocol-buffers/docs/encoding#varints

###The main header

1 byte
The first byte of the TWKB-geometry only apears once, no matter what type it is.

That byte is used like this:

bit 1	**Is there an ID?**
* set: The TWKB or subgeometries have ID, see the specific types
* unset: No ID and no space for ID

bit 2-4 **serialization method:** describe what serialisation method is used. See section "Delta value array rules"	<br>
bit 5-8 **precision:** tells how many decimals to use in coordinates see section "Storage of coordinates"
	

###The type

* UINT8 holding geometry **type** and **number of dimensions**

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

This can be solved in a lot of ways, each one with it's pros and cons. In the first TWKB-byte, bit nr 2-4 describes which serialisation method that is used. <br>
That gives only 8 possibilities, but that will have to do for now

### Method 1 <br>
This is the default now <br>
Also here the first coordinate is stored as full value and the ones after that as delta values<br>
The difference is that here all values are stored as signed varInt with a maximum of 8 bytes<br><br>
This seems to be the most promising method.
The method is described here:
https://developers.google.com/protocol-buffers/docs/encoding#varints

Probably multiple compression methods will not be needed if there is no very good reason. 
Then we have other plans for the three bits in the header.

