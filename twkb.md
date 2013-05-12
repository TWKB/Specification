#TWKB

First Draft 
version 0.01

Specification for "Tiny WKB", TWKB
	
##Abstract 

TWKB is meant to be a multi purpose slimmed format for transporting vector geometry data.
	
##The structure

###The main header

1 byte
The first byte of the TWKB-geometry only apears once, no matter what type it is.

That byte is used like this:

bit 1	**endianess**
* set: little endian
* unset: big endian

bit 2-4 **serialization method:** describe what serialisation method is used. See section " Delta value array rules"	
bit 5-8 **precission:** tells how many decimals to use in coordinates see section Storage of coordinates
	

###The type

1 byte holding geometry **type** and **number of dimmensions**

bit 1-5 gives 31 type positions, we use a few of them:

* 1	Point (1 single point)
* 2	Linestring
* 3	Polygon
* 4	MultiPoint
* 5	MultiLinestring
* 6	MultiPolygon
* 7	GeometryCollection
* 20	PointArray for use in topogeometries  (edges) 
* 21	PointArrayIndex
* 22	TopoLinestring
* 23	TopoPolygon

bit 6-8; number of dimmensions (ndims)

###Description type by type

#### Type 1, **Point**
UINT32 holding the id of the Point

* If this is top level of TWKB:<br>
	coordinates as 4 byte integers:<br>
		INT32 x ndims
* If this is a nested point (Multipoint or GeometryCollection):<br>
	follows delta value array rules, see below
	
#### Type 2, Linestring
UINT32 holding the id of the Linestring
UINT32 npoints
	a 4 byte integer holding number of vertex-points in linestrings
	
PointArray:
* If this is top level of TWKB:<br>
	first vertex point coordinates as 4 byte integers:<br>
		INT32 x ndims<br>
	the rest follows delta value array rules, see below	
* If this is a nested Linestring (MultiLinestring or GeometryCollection)<br>
	follows delta value array rules, see below
	
#### Type 3, Polygon
UINT32 holding the id of the Polygon
UINT32 nrings
	a 4 byte integer holding number of rings in the polygon (first ring is boundary, the rest is holes)
	
for each ring
{
UINT32 npoints
	a 4 byte integer holding number of vertex-points in polygon ring
	
PointArray:
If this is top level of TWKB and first ring:
	first vertex point coordinates as 4 byte integers:
		INT32 x ndims
	the rest follows delta value array rules (see below)		
If this is a hole or nested Polygon (MultiPolygon or GeometryCollection
	follows delta value array rules (see below)
}	

#### Type 4, MultiPoint
UINT32 npoints
	a 4 byte integer holding number of points in the MultiPoint

For each Point
{
UINT32 holding the id of the Point
If this is top level of TWKB:
	first  point coordinates as 4 byte integers:
		INT32 x ndims
	the rest follows delta value array rules (see below)		
If this is a nested MultiPoint (GeometryCollection)
	follows delta value array rules (see below)	
}	


#### Type 5, MultiLineString
UINT32 nlinestrings
	a 4 byte integer holding number of Linestrings in the the MultiLinestring

For each Linestring
{

see type 2 Linestring
}	


#### Type 6, MultiPolygon
UINT32 npolygons
	a 4 byte integer holding number of polygons in the the Multipolygon

For each polygon
{
see type 3 polygon
}	

#### Type 7, GeoemtryCollection	
UINT32 ngeometries
	a 4 byte integer holding number of subgeoemtries
for each geometry
{
1 byte describing type and number of dimmensions

see description for respective type
}
	
	
#### Type 22	topo linestring
UINT32 id
	the id of the linestring
UINT16 narrays
	a 4 byte integer holding number of pointarrays used to build the linestring

array of id-values to linestrings or points (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

#### Type 23	topo polygon
UINT32 id
	the id of the polygon
UINT16 narrays
	a 4 byte integer holding number of pointarrays used to build the polygon

array of id-values to linestrings or points (those linestrings or points can be a part of this twkb-geom or another, it is up to the client to index the points and linestrings for fast find)

## Storage of coordinates
All storage is integers. So what happens is that the value gets multiplied with 10^precission-value, and is then rounded to closest integer
when reading the twkb, the value should be divided with 10^precission-value

So if the precission value is 2, we multiply the value with 100 and rounds the result to closest integer when we create out twkb-geometry and do the reveresed operation when we read it.

## Delta value array rules
This is about how the coordinates are serialized. The problem to be solved is that we want to use as little space as possible to store out coordinates.
To do that we cannot just use a fixed size because than we have to use the biggest size possible nessecary. Instead we have to find a way to change the storage size as the need changes. 
The delta value between two coordinates vary a lot.

This can be solved in a lot oof ways, each one with it's pros and cons. In the first TWKB-byte, bit nr 2-4 describes which serialisation method that is used. 
That gives only 8 possibilities, but that will have to do for now

## method nr 0 
This method is tested.  It seems fast and quite compressed. 

as seen from the clients perspective :
Reading the first point array in the twkb-geometry

1)	first read INT32 x "number of dimmensions"
		that is the first point described with full coordinates

2)	read one INT8
		there is two possibilities:
		a)		the value is between -127 and 127
					That is our delta value. The difference between the first point first dimmension and the second point first dimmension.
		b)		the value is -127 (or binary value 11111111 on most systems), it is a flag of that the coordinate didn't fit in INT8. 
					then read another INT8, the value of that is telling what size that is used instead. The value referes to number of bytes, so 1 is INT8, 2 is INT16 and 4 is INT32
						then we can read our coordinate using that size. That new size is now the current size and will be used until we meet a new "change in size flag"

3)	use the last used size to read again.
		if the value is the lowest possible number with the current size it is a change in size, and the next INT8 will tell what size to be used.
		
		
		
Same thing in other words:

1)	first coordinate is stored with 1 INT32 per dimmension
2)	next one is always INT8 giving a delta value or signaling a change in size
3)	Changes in size is signaled by the lowest possible number that the storage size can hold:
		INT8 -> -128	
		INT16 -> -32768
		INT32 -> -2147483648
		
		this can be evaluted from:
		INT8 -> -1<<7
		INT16 -> -1<<15
		INT32 -> -1<<31
4)	first byte after "size change flag" tells the new current flag:
		1 -> INT8
		2 -> INT16
		3 -> INT32
		
5)	after a change that new value is valid until a new "size change flag" is met


#### method nr 1 
not tested
The same as method 0 but have 4 sizes instead of 3:
INT8
INT16
INT32
INT64

The first coordinate is using INT64 instead of INT32

#### method nr 2
not tested
Similar as method 0 but each dimmensions holds its own sizes. 
So a change in size for latitude doesn't affect the size we use for longitude.

compared to method 0:
pros:
1)	for example the northern coastline of an iland will have bigger deltas on lon than on lat. With this method we can store those dimmensions on different size.
2)	If we use one dimmension for something else than spatial information like temperatures, the correlation between the need of size between different dimmensions is very bad.

cons:
More sizechanges will be needed which takes some space, and the code will be a little bit more complicated which might makes it slightly slower


#### method nr 2 (should probably not be in the spec at all)
tested
map all the sizes, 2 bits per coordinate-value before the delta-values
Those 2 bits can tell what size to use

This is tested and was slightly slower and produced a littlle bigger result on the tested datasets.

If there is a lot of chages in the need of storage-size this might be a good choice.
	