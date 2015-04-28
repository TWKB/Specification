# "Tiny Well-known Binary" or "TWKB"

| Version | Release     |
| ------- | ----------- |
| 0.21    | May 1, 2015 |
	
## Abstract 

TWKB is a multi-purpose format for serializing vector geometry data into a byte buffer, with an emphasis on minimizing size of the buffer.
	
### Why not WKB? 

The original OGC "well-known binary" format is a simple format, and is capable of easily representing complex OGC geometries like nested collections, but it has two important drawbacks for use as a production serialization: 

* it is not aligned, so it doesn't support efficient direct memory access; and,
* it uses IEEE doubles as the coordinate storage format, so for data with lots of spatially adjacent coordinates (basically, all GIS data) it wastes a lot of space on redundant specification of coordinate information.

A new serialization format can address the problem of alignment, or the problem of size, but not both. Given that most current client/server perfomance issues are bottlenecked on network transport times, TWKB concentrates on solving the problem of serialization size.

### Basic Principles

TWKB applies the following principles:

* Only store the absolute position once, and store all other positions as delta values relative to the preceding position.
* Only use as much address space as is necessary for any given value. Practically this means that "variable length integers" or "[varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)" are used throughout the specification for storing values in any situation where numbers greater than 128 might be encountered.


## Structure

### Primitives Types and Group Typess

TWKB supports two different use cases with different structures:

* Primitive types, that have exact analogues in WKB. For example, Point, LineString, MultiPoint and MultiLinestring. Each primitive geometry usually comes from a single feature in the original data, either a single row in a database, or a single record in a GIS file. Each primitive geometry has only one (optional) unique identifer that ties it back to the original data.

* Group types, that allow geometries from multiple features to be serialized into a single byte buffer. Each *element* of a group generally carries a unique identifier that allows it to be tied back to a unique feature in the orignal data. Groups allow collections of features to be serialized into a single binary to ship to a remote client.

### Standard Attributes

Every TWKB geometry contains standard attributes at the top of the object.

* A **metadata header** to indicate which optional attributes to expect, and the storage precision of all coordinates in the geometry.
* An optional **size** in bytes of the object.
* A **type number** and **dimensionality** byte to describe the OGC geometry type.
* An optional **bounding box** of the geometry.
* An optional **unique integer identifier** of the geometry.

#### Metadata Header

**Size:** 1 byte

The first byte of TWKB is mandatory, and encodes the following information:

| Bits    | Role           | Purpose                                         | 
| ------- | -------------- | ----------------------------------------------- |
| 1       | Boolean        | Is there an ID attribute?                       |
| 2       | Boolean        | Is there a size attribute?                      |
| 3       | Boolean        | Is there a bounding box?                        |
| 5-8     | Signed Integer | What precision should coordinates be stored at? |

	
#### Size [Optional]

**Size:** 1 varInt (so, variable)

If the size attribute bit is set in the metadata header, a varInt with size infoformation comes next. The values is the size in bytes of the remainder of the geometry after the size attribute.

When encountered in collections, an application can use the size attribute to advance the read pointer the the start of the next geometry. This can be used for a quick scan through a set of geometries, for reading just bounding boxes, or to distibute the read process to different threads.

#### Type & Dimensions

**Size:** 1 byte, holding geometry **type** and **number of dimensions**

The type-byte stores both the geometry type, and the dimensionality of the coordinates of the geometry. 

* Bits 1-5 store the geometry type (there is space for 31 and we currently use 9):

  * 1	Point
  * 2	Linestring
  * 3	Polygon
  * 4	MultiPoint
  * 5	MultiLinestring
  * 6	MultiPolygon
  * 7	GeometryCollection
  * 20	Homogeneous Group
  * 21	Heterogeneous Group

* Bits 6-7 store the number of coordinate dimensions. The coordinate dimension number is interpreted as:

  * 0 X/Y
  * 1 X/Y/Z
  * 2 X/Y/M
  * 3 X/Y/Z/M
  
The geometry type can be read by masking out the lower five bits (`type & 0x1f`).

The coordinate dimension can be read by masking out bits 6-7 and shifting them down (`(type & 0x60) >> 5`).


#### Bounding Box [Optional]

2 varInt per dimmension<br> 
If the bounding box bit in the first byte is set a bounding box comes next.
A bounding box is represented with varInts like:<br>
xMin, deltax, ymin, deltay, .......
This way a 2d box can be read without knowing the number of dimmensions (and then juming to next geoemtry. If the geoemtries is to be read all dimmensions of the bbox must of course be read.)


### Description of Types

#### Type 1, Point

* varInt **ID**, optional, only used if very first bit of TWKB is set
* the point coordinates (as deltavalue if in a MultiPoint or Geometry Collection)
	
    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    coordinates       varint[]
  
  
#### Type 2, Linestring

* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npoints           varint
    coordinates       varint[]

#### Type 3, Polygon

* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **nrings** holding number of rings (first ring is boundary, the others are holes)

For each ring {<br>
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    nrings            varint
    npoints[0]        varint
    coordinates[0]    varint[]
    ...
    npoints[n]        varint
    coordinates[n]    varint[]


#### Type 4, MultiPoint (with one id for all)
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **npoints** holding number of points
* a Point Array see section "Delta value array rules" below

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npoints           varint
    coordinates       varint[]


#### Type 5, MultiLineString (with one id for all)
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **nlinestrings**  holding number of linestrings

For each linestring {<br>
* varInt **npoints** holding number of vertex-points
* a Point Array see section "Delta value array rules" below<br>
}	

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    nlinestrings      varint
    npoints[0]        varint
    coordinates[0]    varint[]
    ...
    npoints[n]        varint
    coordinates[n]    varint[]


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

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npolygons         varint
    nrings[0]         varint
    npoints[0][0]     varint
    coordinates[0][0] varint[]
    ...
    nrings[n]         varint
    npoints[n][m]     varint
    coordinates[n][m] varint[]


#### Type 7, GeometryCollection 
* varInt **ID**, optional, only used if very first bit of TWKB is set
* varInt **ngeometries** holding number of geometries

For each geometry {<br>
* varInt describing type and ndim  of subgeometry<br>
* a geometry of the specified type without ID<br>
}

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    ngeometries       varint
    geom              twkb[]


#### Type 20, Homogeneous Group

* varInt **npoints** holding number of points

For each point {<br>
* Point of type 1
}


#### Type 21, Heterogeneous Group




## Coordinate Storage

All storage is **integers**. So what happens is that the value gets multiplied with 10^precision-value, and is then rounded to closest integer<br>When reading the twkb, the value should be divided with 10^precision-value

So if the precision value is 2, we multiply the value with 100 and rounds the result to closest integer when we create out twkb-geometry and do the reveresed operation when we read it.

## Delta value array rules

This is about how the coordinates are serialized. The problem to be solved is that we want to use as little space as possible to store our coordinates.<br>
To do that we cannot just use a fixed data type because than we have to use the biggest size possible necessary. Instead we have to find a way to change the storage size as the need changes. <br>
The delta value between two coordinates vary a lot.<br>
This is solved by encoding the values as varInt
The method is described here:
https://developers.google.com/protocol-buffers/docs/encoding#varints



