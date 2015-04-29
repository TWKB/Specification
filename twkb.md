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

The "precision" refers to the number of base-10 decimal places stored. 

* A **positive** precision implies retaining information to the right of the decimal place

    * 41231.1231 at precision=2 would store 41231.12  
    * 41231.1231 at precision=1 would store 41231.1  
    * 41231.1231 at precision=0 would store 41231

* A **negative** precision implies rouding up to the left of the decimal place

    * 41231.1231 at precision=-1 would store 41230  
    * 41231.1231 at precision=-2 would store 41200  

In order to support negative precisions, the precision number should be stored using zig-zag encoding (see "ZigZag Encode" below).
	
    
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

**Size:** 2 varints per dimension

Each dimension of a bounding box is represented by a pair of varints: 

* the minimum of the dimension
* the relative maximum of the dimension (relative to the minimum)

So, for example:

* [xmin, deltax, ymin, deltay, zmin, deltaz]


## Description of Types

### PointArrays

Every object contains a point array, which is an array of coordinates, stored as varints. The final values in the point arrays are a result of four steps:

* convert doubles to integers
* calculate integer delta values between successive points
* zig-zag encode delta values to move negative values into positive range
* varint encode the final value

The result is a very compact representation of the coordinates. 

#### Convert to Integers

All storage using varints, which are **integers**. However, before being encoded most coordinate are doubles. How are the double coordinates converted into integers? And how are the integers converted ino the final array of varints?

Each coordinate is multiplied by the **geometry precision** value (from the metadata header), and then rounded to the nearest integer value (`round(doubecoord/precision)`). When converting from TWKB back to double precision, the reverse process is applied.

#### Calculate Delta Values

Rather than storing the absolute position of every vertex, we store the **difference** between each coordinate and the previous coordinate of that dimension. So the coordinate values of a point array are:

          x1,        y1
    (x2 - x1), (y2 - y1)
    (x3 - x2), (y3 - y2)
    (x4 - x3), (y4 - y2)
    ...
    (xn - xn-1), (yn - yn-1)

Delta values of integer coordinates are nice and small, and they store very compactly as varints.


#### ZigZag Encode

The varint scheme for indicating a value larger than one byte involves setting the "high bit", which is the same thing that standard integers use for storing negative numbers. 

So negative numbers need to first be encoded using a "[zig zag](https://developers.google.com/protocol-buffers/docs/encoding)" encoding, which keep absolute values small, but do not set the high bit as part of encoding negative numbers.

The effect is to convert small negative numbers into small positive numbers, which is exactly what we want, since delta values will tend to be small negative and positive numbers:

    -1 => 1
     1 => 2
    -2 => 3
     2 => 4

A computationally fast formula to generate the zig-zag value is 

    /* for 32-bit signed integer n */
    unsigned int zz = (n << 1) ^ (n >> 31)
    
    /* for 64-bit signed integer n */
    unsigned long zz = (n << 1) ^ (n >> 63)


#### VarInt Encode

Variable length integers are a clever way of only using as many bytes as necessary to store a value. The method is described here:

https://developers.google.com/protocol-buffers/docs/encoding#varints

The varint scheme sets the high bit of a byte as a flag to indicate when more bytes are needed to fully represent a number. Decoding a varint accumulates the information in each flagged byte until an unflagged byte is found and the integer is complete and ready to return.


### Primitive Types

#### Point [Type 1]

Because **points** only have one coordinate, the coordinates will be the true values (not relative) except in cases where the point is embedded in a collection.

Bounding boxes are permitted on points, but **discouraged** since they just duplicate the values already available in the point coordinate. Similarly, unless the point is part of a collection (where random access is a possible use case), the size should also be omitted.

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    pointarray        varint[]
  
  
#### Linestring [Type 2]

A **linestring** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **npoints** varint giving the number of points in the linestring
* if **npoints** is zero, the linestring is "empty", and there is no further content

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npoints           varint
    pointarray        varint[]


#### Polygon [Type 3]

A **polygon** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **nrings** varint giving the number of rings in the polygon
* if **nrings** is zero, the polygon is "empty", and there is no further content
* for each ring there will be

    * an **npoints** varint giving the number of points in the ring
    * a pointarray of varints
    * rings are assumed to be implicitly closed, so the first and last point should not be the same

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    nrings            varint
    npoints[0]        varint
    pointarray[0]     varint[]
    ...
    npoints[n]        varint
    pointarray[n]     varint[]


#### MultiPoint [Type 4]

A **multipoint** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **npoints** varint giving the number of points in the multipoint
* if **npoints** is zero, the multipoint is "empty", and there is no further content

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npoints           varint
    pointarray        varint[]


#### MultiLineString [Type 5]

A **multilinestring** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **nlinestrings** varint giving the number of linestrings in the collection
* if **nlinestrings** is zero, the collection is "empty", and there is no further content
* for each linestring there will be

    * an **npoints** varint giving the number of points in the linestring
    * a pointarray of varints

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    nlinestrings      varint
    npoints[0]        varint
    pointarray[0]     varint[]
    ...
    npoints[n]        varint
    pointarray[n]     varint[]


#### MultiPolygon [Type 6]

A **multipolygon** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **npolygons** varint giving the number of polygons in the collection
* if **npolygons** is zero, the collection is "empty", and there is no further content
* for each polygon there will be

    * an **nrings** varint giving the number of rings in the linestring
    * for each ring there will be

        * an **npoints** varint giving the number of points in the ring
        * a pointarray of varints
        * rings are assumed to be implicitly closed, so the first and last point should not be the same

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    npolygons         varint
    nrings[0]         varint
    npoints[0][0]     varint
    pointarray[0][0]  varint[]
    ...
    nrings[n]         varint
    npoints[n][m]     varint
    pointarray[n][m]  varint[]


#### GeometryCollection [Type 7]

A **geometrycollection** has, in addition to the standard metadata:

* an optional "id" (if indicated in the metadata header)
* an **ngeometries** varint giving the number of geometries in the collection
* for each geometry there will be a complete TWKB geometry, readable using the rules set out above

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    [id]              varint
    ngeometries       varint
    geom              twkb[]

### Group Types

#### Homogeneous Group [Type 20]

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    subtype           varint
    [bounds]          bbox
    ngeometries       varint
    stripped_geom     twkb[]

A "stripped" geometry is a primitive geometry with everything prior to the "id" field removed. Stripped geometries in groups always have an "id" field, it is not optional.


#### Heterogeneous Group [Type 21]

The layout is:

    metadata_header   byte
    [size]            varint
    type_and_dims     byte
    [bounds]          bbox
    ngeometries       varint
    reduced_geom     twkb[]

A "reduced" geometry is a primitive geometry with everything prior to the "id" field removed except the type field. Reduced geometries in groups always have an "id" field, it is not optional.





