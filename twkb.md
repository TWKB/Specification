# "Tiny Well-known Binary" or "TWKB"

| Version | Release     |
| ------- | ----------- |
| 0.23    | May 1, 2015 |
	
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

### Standard Attributes

Every TWKB geometry contains standard attributes at the top of the object.

* A **metadata header** to indicate which optional attributes to expect, and the storage precision of all coordinates in the geometry.
* A **type number** and **precision** byte to describe the OGC geometry type.
* An optional **extended dimension** byte with information about existence and presence of Z & M dimensions.
* An optional **size** in bytes of the object.
* An optional **bounding box** of the geometry.
* An optional **unique integer array** of sub-components for multi-geometries.


#### Type & Precision

**Size:** 1 byte, holding geometry **type** and **number of dimensions**

The type-byte stores both the geometry type, and the dimensionality of the coordinates of the geometry. 

| Bits    | Role             | Purpose                                         | 
| ------- | ---------------- | ----------------------------------------------- |
| 1-4     | Unsigned Integer | Geometry type                                   |
| 5-8     | Signed Integer   | Geometry precision                              |

* Bits 1-4 store the geometry type (there is space for 16 and we currently use 7):

    * 1 Point
    * 2 Linestring
    * 3 Polygon
    * 4 MultiPoint
    * 5 MultiLinestring
    * 6 MultiPolygon
    * 7 GeometryCollection

* Bits 5-8 store the "precision", which refers to the number of base-10 decimal places stored. 

    * A **positive** precision implies retaining information to the right of the decimal place

        * 41231.1231 at precision=2 would store 41231.12
        * 41231.1231 at precision=1 would store 41231.1
        * 41231.1231 at precision=0 would store 41231

    * A **negative** precision implies rouding up to the left of the decimal place

        * 41231.1231 at precision=-1 would store 41230  
        * 41231.1231 at precision=-2 would store 41200  

In order to support negative precisions, the precision number should be stored using zig-zag encoding (see "ZigZag Encode" below).

* The geometry type can be read by masking out the lower four bits (`type & 0x0F`).
* The precision can be read by masking out the top four bits (`(type & 0xF0) >> 4`).

  

#### Metadata Header

**Size:** 1 byte

The metadata byte of TWKB is mandatory, and encodes the following information:

| Bits    | Role           | Purpose                                         | 
| ------- | -------------- | ----------------------------------------------- |
| 1       | Boolean        | Is there a bounding box?                        |
| 2       | Boolean        | Is there a size attribute?                      |
| 3       | Boolean        | Is there an ID list?                            |
| 4       | Boolean        | Is there extended precision information?        |
| 5       | Boolean        | Is this an empty geometry?                      |
| 6-8     | Boolean        | Unused                                          |


    
#### Extended Dimensions [Optional]

**Size:** 1 byte

For some coordinate reference systems, and dimension combinations, it makes no sense to store all dimensions using the same precision. 

For example, for data with X/Y/Z/T dimensions, using a geographic coordinate system, it might make sense to store the X/Y (longitude/latitude) dimensions with precision of 6 (about 10 cm), the Z with a precision of 1 (also 10 cm), and the T with a precision of 0 (whole seconds). A single precision number cannot handle this case, so the extended precision slot is optionally available for it.

The extended precision byte holds:

| Bits    | Role           | Purpose                                         | 
| ------- | -------------- | ----------------------------------------------- |
| 1       | Boolean        | Geometry has Z coordinates?                     |
| 2       | Boolean        | Geometry has M coordinates?                     |
| 3-5     | Integer        | Precision for Z coordinates.                    |
| 6-8     | Integer        | Precision for M coordinates.                    |

The extended precision values are always positive (only deal with digits to the left of the decimal point)

* The presence of Z can be read by masking out bit 1: `(extended & 0x01)`
* The presence of M can be read by masking out bit 2: `(extended & 0x02)`
* The value of Z precision can be read by masking and shifting bits 3-5: `(extended & 0x1C) >> 2`
* The value of M precision can be read by masking and shifting bits 6-8: `(extended & 0xE0) >> 5`

	
#### Size [Optional]

**Size:** 1 varint (so, variable size)

If the size attribute bit is set in the metadata header, a varInt with size infoformation comes next. The values is the size in bytes of the remainder of the geometry after the size attribute.

When encountered in collections, an application can use the size attribute to advance the read pointer the the start of the next geometry. This can be used for a quick scan through a set of geometries, for reading just bounding boxes, or to distibute the read process to different threads.


#### Bounding Box [Optional]

**Size:** 2 varints per dimension (also variable size)

Each dimension of a bounding box is represented by a pair of varints: 

* the minimum of the dimension
* the relative maximum of the dimension (relative to the minimum)

So, for example:

* [xmin, deltax, ymin, deltay, zmin, deltaz]


#### ID List [Optional]

**Size:** N varints, one per sub-geometry

The TWKB collection types (multipoint, multilinestring, multipolygon, geometrycollection)  

* can be used as single values (a given row contains a single collection object), or 
* can be used as aggregate wrappers for values from multiple rows (a single object contains geometries read from multiple rows).

In the latter case, it makes sense to include a unique identifier for each sub-geometry that is being wrapped into the collection. The "idlist" attribute is an array of varint that has one varint for each sub-geometry in the collection.


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

Each coordinate is multiplied by the **geometry precision** value (from the metadata header), and then rounded to the nearest integer value (`round(doubecoord/10^precision)`). When converting from TWKB back to double precision, the reverse process is applied.

#### Calculate Delta Values

Rather than storing the absolute position of every vertex, we store the **difference** between each coordinate and the previous coordinate of that dimension. So the coordinate values of a point array are:

          x1,        y1
    (x2 - x1), (y2 - y1)
    (x3 - x2), (y3 - y2)
    (x4 - x3), (y4 - y2)
    ...
    (xn - xn-1), (yn - yn-1)

Delta values of integer coordinates are nice and small, and they store very compactly as varints.

Every coordinate value in a geometry **except the very first one** is a delta value relative to the previous value processed. Examples:

* The second coordinate of a line string should be a delta relative to the first.
* The first coordinate of the second ring of a polygon should be a delta relative to the last coordinate of the first ring. 
* The first coordinate of the third linestring in a multilinestring should be a delta relative to the last coordinate of the second linestring.

Basically, implementors should always keep the value of the previous coordinate in hand while processing a geometry to use to difference against the incoming coordinate. By setting the very first "previous coordinate" value to [0,0] it is possible to use the same differencing code throughout, since the first value processed will thus receive a "delta" value of itself.

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

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    pointarray        varint[]
  
  
#### Linestring [Type 2]

A **linestring** has, in addition to the standard metadata:

* an **npoints** varint giving the number of points in the linestring
* if **npoints** is zero, the linestring is "empty", and there is no further content

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    npoints           varint
    pointarray        varint[]


#### Polygon [Type 3]

A **polygon** has, in addition to the standard metadata:

* an **nrings** varint giving the number of rings in the polygon
* if **nrings** is zero, the polygon is "empty", and there is no further content
* for each ring there will be
    * an **npoints** varint giving the number of points in the ring
    * a pointarray of varints
    * rings are assumed to be implicitly closed, so the first and last point should not be the same

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    nrings            varint
    npoints[0]        varint
    pointarray[0]     varint[]
    ...
    npoints[n]        varint
    pointarray[n]     varint[]


#### MultiPoint [Type 4]

A **multipoint** has, in addition to the standard metadata:

* an optional "idlist" (if indicated in the metadata header)
* an **npoints** varint giving the number of points in the multipoint
* if **npoints** is zero, the multipoint is "empty", and there is no further content

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    npoints           varint
    [idlist]          varint[]
    pointarray        varint[]


#### MultiLineString [Type 5]

A **multilinestring** has, in addition to the standard metadata:

* an optional "idlist" (if indicated in the metadata header)
* an **nlinestrings** varint giving the number of linestrings in the collection
* if **nlinestrings** is zero, the collection is "empty", and there is no further content
* for each linestring there will be
    * an **npoints** varint giving the number of points in the linestring
    * a pointarray of varints

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    nlinestrings      varint
    [idlist]          varint[]
    npoints[0]        varint
    pointarray[0]     varint[]
    ...
    npoints[n]        varint
    pointarray[n]     varint[]


#### MultiPolygon [Type 6]

A **multipolygon** has, in addition to the standard metadata:

* an optional "idlist" (if indicated in the metadata header)
* an **npolygons** varint giving the number of polygons in the collection
* if **npolygons** is zero, the collection is "empty", and there is no further content
* for each polygon there will be
    * an **nrings** varint giving the number of rings in the linestring
    * for each ring there will be
        * an **npoints** varint giving the number of points in the ring
        * a pointarray of varints
        * rings are assumed to be implicitly closed, so the first and last point should not be the same

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    npolygons         varint
    [idlist]          varint[]
    nrings[0]         varint
    npoints[0][0]     varint
    pointarray[0][0]  varint[]
    ...
    nrings[n]         varint
    npoints[n][m]     varint
    pointarray[n][m]  varint[]


#### GeometryCollection [Type 7]

A **geometrycollection** has, in addition to the standard metadata:

* an optional "idlist" (if indicated in the metadata header)
* an **ngeometries** varint giving the number of geometries in the collection
* for each geometry there will be a complete TWKB geometry, readable using the rules set out above

The layout is:

    type_and_dims     byte
    metadata_header   byte
    [extended_dims]   byte
    [size]            varint
    [bounds]          bbox
    ngeometries       varint
    [idlist]          varint[]
    geom              twkb[]







