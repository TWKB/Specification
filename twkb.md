#TWKB

First Draft <br>
version 0.05

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

bit 1	**endianess**
* set: little endian
* unset: big endian

bit 2-4 **serialization method:** describe what serialisation method is used. See section "Delta value array rules"	<br>
bit 5-8 **precission:** tells how many decimals to use in coordinates see section "Storage of coordinates"
	

###The type

* UINT8 holding geometry **type** and **number of dimmensions**

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

bit 6-8:  number of dimmensions (ndims)

###Description type by type

#### Type 1, **Point**
varInt holding the id of the Point

the point coordinates (as deltavalue if in a MultiPoint or Geometry Collection)
	
#### Type 2, Linestring
* varInt **ID**
* varInt **npoints** a 4 byte integer holding number of vertex-points
* a Point Array see section "Delta value array rules" below

#### Type 3, Polygon
* varInt **ID**
* varInt **nrings** a 4 byte integer holding number of rings (first ring is boundary, the rest is holes)

For each ring{<br>
* varInt **npoints** a 4 byte integer holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

#### Type 4, MultiPoint (with one id for all)
* varInt **ID**
* varInt **npoints** a 4 byte integer holding number of points
* a Point Array see section "Delta value array rules" below

#### Type 5, MultiLineString (with one id for all)
* varInt **ID**
* varInt **nlinestrings** a 4 byte integer holding number of linestrings

For each linestring{<br>
* varInt **npoints** a 4 byte integer holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

#### Type 6, MultiPolygon (with one id for all)
* varInt **ID**
* varInt **npolygons** a 4 byte integer holding number of polygons

For each polygon{<br>
* varInt **nrings** a 4 byte integer holding number of rings (first ring is boundary, the rest is holes)

For each ring{<br>
* varInt **npoints** a 4 byte integer holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	<br>
}	

#### Type 7, GeometryCollection 
* varInt **ID**
* varInt **ngeometries** a 4 byte integer holding number of geometries

For each geometry{<br>
* varInt describing type and ndim  of subgeometry<br>
* a geometry of the specified type without ID<br>
}

#### Type 21, MultiPoint (with individual id)

* varInt **npoints** a 4 byte integer holding number of points

For each point{<br>
* Point of type 1
}


#### Type 22, MultiLineString (with individual id)

* varInt **nlinestrings** a 4 byte integer holding number of linestrings

For each linestring{<br>
* Linestring of type 2
}

#### Type 23, MultiPolygon (with individual id)

* varInt **npolygons** a 4 byte integer holding number of polygons

For each polygon{<br>
* Polygon of type 3
}

#### Type 24, MultiGometryCollection (with individual id)

* varInt **collections** a 4 byte integer holding number of collections

For each collection [MultiPoints, MultiLinestrings, MultiPolygons or GeometryCollections){<br>
* MultiPoint of type 4
or
* MultiLinestrings of type 5
or 
* MultiPolygon of type 6
or
* GeometryCollection of type 7
}

#### Type 25	topo linestring
* varInt **ID**
* varInt **ncomponents** a 2 byte integer holding number of components used to build the linestring
* array of id-values to linestrings or points (type 1,2 or members of type 7, 21 or 22) (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

#### Type 26	topo polygon
* varInt **ID**
* varInt **ncomponents** a 2 byte integer holding number of components used to build the polygon
* array of id-values to linestrings or points (type 1,2 or members of type 7, 21 or 22) (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

## Storage of coordinates

All storage is **integers**. So what happens is that the value gets multiplied with 10^precission-value, and is then rounded to closest integer<br>
when reading the twkb, the value should be divided with 10^precission-value

So if the precission value is 2, we multiply the value with 100 and rounds the result to closest integer when we create out twkb-geometry and do the reveresed operation when we read it.

## Delta value array rules

This is about how the coordinates are serialized. The problem to be solved is that we want to use as little space as possible to store our coordinates.<br>
To do that we cannot just use a fixed data type because than we have to use the biggest size possible nessecary. Instead we have to find a way to change the storage size as the need changes. <br>
The delta value between two coordinates vary a lot.<br>

This can be solved in a lot of ways, each one with it's pros and cons. In the first TWKB-byte, bit nr 2-4 describes which serialisation method that is used. <br>
That gives only 8 possibilities, but that will have to do for now

### method nr 0 <br>
This method is tested.  It seems fast and quite compressed. 

as seen from the clients perspective :<br>
Reading the first point array in the twkb-geometry<br>

<ol>
<li>first read INT32 x "number of dimmensions"<br>
		that is the first point described with full coordinates</li>
<li>read one INT8<br>
		there is two possibilities:
		<ol>
		<li>	the value is between -127 and 127<br>
					That is our delta value. The difference between the first point first dimmension and the second point first dimmension.</li>
		<li>the value is -127 (or binary value 11111111 on most systems), it is a flag of that the coordinate didn't fit in INT8. <br>
					then read another INT8, the value of that is telling what size that is used instead. The value referes to number of bytes, so 1 is INT8, 2 is INT16 and 4 is INT32<br>
						then we can read our coordinate using that size. That new size is now the current size and will be used until we meet a new "change in size flag"</li>
		</ol>
</li>
<li>use the last used size to read again.<br>
		if the value is the lowest possible number with the current size it is a change in size, and the next INT8 will tell what size to be used.</li>
</ol>
		
		
		
####Same thing in other words:

<ol>
<li>first coordinate is stored with 1 INT32 per dimmension</li>
<li>next one is always INT8 giving a delta value or signaling a change in size</li>
<li>Changes in size is signaled by the lowest possible number that the storage size can hold:
	<ul>
		<li>INT8 -> -128</li>
		<li>INT16 -> -32768</li>
		<li>INT32 -> -2147483648</li>
	</ul>
		this can be evaluted from:
	<ul>
		<li>INT8 -> -1<<7</li>
		<li>INT16 -> -1<<15</li>
		<li>INT32 -> -1<<31</li>
	</ul>
</li>
<li>first byte after "size change flag" tells the new current flag:
	<ul>
		<li>1-> INT8</li>
		<li>2 -> INT16</li>
		<li>3 -> INT32</li>
	</ul>
</li>		
<li>after a change that new value is valid until a new "size change flag" is met</li>
</ol>


### method nr 1  <br>
tested <br>
Also here the first coordinate is stored as full value and the ones after that as delta values<br>
The difference is that here all values is stored as signed varInt with of maximum 8 bytes<br><br>
This seems to be the most promising method.

### method nr 2 <br>
not tested <br>
Similar as method 0 but each dimmensions holds its own sizes.  <br>
So a change in size for latitude doesn't affect the size we use for longitude. <br>

compared to method 0: <br>
pros: <br>
* for example the northern coastline of an iland will have bigger deltas on lon than on lat. With this method we can store those dimmensions on different size.
* If we use one dimmension for something else than spatial information like temperatures, the correlation between the need of size between different dimmensions is very bad.
* For pointclouds this method will be superior since totally different type of data will be stored as dimmensions. Then those dimmensions needs there own storage handling.

cons: <br>
* More sizechanges will be needed which takes some space, and the code will be a little bit more complicated which might makes it slightly slower

For this method to work properly it is important to also include some mechanism that "looks forward" so the decrease of size not is done when the next coordinate will force increasing it again.</br>
There is ideas for a "rolling cache" that can handle this

### method nr 3 (should probably not be in the spec at all) <br>
tested <br>
map all the sizes, 2 bits per coordinate-value before the delta-values <br>
Those 2 bits can tell what size to use <br>

This is tested and was slightly slower and produced a littlle bigger result on the tested datasets.

If there is a lot of chages in the need of storage-size this might be a good choice.
	
