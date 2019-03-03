# X3 Lossless Audio Compression

X3 is a simple and effective lossless audio compressor for low entropy sound. It is based on
[Shorten](<https://en.wikipedia.org/wiki/Shorten_(file_format)>) and has some of the features
of [FLAC](https://xiph.org/flac/) but is designed specifically for use in underwater sound
recording tags and buoys. It is much faster than FLAC, but does not acheive the compression
ratio. The name comes from the compression factor that it usually achieves, i.e., 3 times.
The algorithm is described in a paper in the [Journal of the Acoustical Society of
America 133:1387-1398, 2013](http://link.aip.org/link/?JAS/133/1387).

This repository is a fork of the work from the [original authors](https://www.soundtags.org/dtags/audio_compression/).

## X3 Data Format

This [paper](http://link.aip.org/link/?JAS/133/1387) describes the protocol in detail. This
only describes the encoded waveform data. It does not describe X3 Archive format (how the
file is structured).

Notes:

- All unsigned 16 and 64 bit integers (u16 and u64) are stored in big-endian format.

**X3 Archive Format**:

```Text
<X3 Archive>
    <Archive Header>
    <Frame>
      <Filter State>
      <Block>
      <Block>
      ...
    <Frame>
      <Filter State>
      <Block>
      <Block>
      ...
    ...
    <Frame>
      <Filter State>
      <Block>
      <Block>
      ...
```

**`<Archive Header>`**

The archive header describes the contents of the file. It contains an ID, and a XML section
described by an embedded XML document.

```Text
Name            Type        Size (bytes)    Description

<Archive Id>    String      8               A fixed value of "X3ARCHIV".
<Frame Header>  Bytes       20              Frame Header that describes the length of <XML MetaData>.
<XML MetaData>  XML String  Variable        An XML string. As described in <XML MetaData>
```

**`<XML MetaData>`**

This describs the data types that will be in the file. Additional metadata frames (and data
types) can be added later in the file as needed.

Note that the XML formatting here is deliberately loose. No classes are given for field contents
and so it is up to the code that will decode the fields to decide how to use the metadata.
This has the advantage of keeping the fields simple and easy to generate by an embedded processor.

Start by defining the file format type and the default data source i.e., the metadata
source with id=0

```XML
  <X3ARCH PROG="x3new.m" VERSION="2.0" />
  <CFG ID="0" FTYPE="XML" />
```

There can be up to 255 other source types in an X3 archive file.
Each data source is associated with a different type of data
and generates a unique output file. This allows multi-rate and
heterogenous data collection, e.g., simultaneous high and low
audio bands, spectral averages, and CTD data.

Add a CFG statement for each source ID - in this case there is
only one data source - an X3-encoded WAV file:

1. select an ID number (1..255) and indicate the file type to produce
   when data from this source is encountered.

   ```XML
     <CFG ID="1" FTYPE="WAV">
   ```

2. notify the sampling rate.

   ```XML
     <FS UNIT="Hz">16000</FS>
   ```

3. suffix to give the output file (doesn't have to be 'wav' - this
   enables multiple audio streams in the same archive.

   ```XML
     <SUFFIX>wav</SUFFIX>
   ```

4. notify the codec that is being used and give the parameters.
   `TYPE`: the name of the encoder.

   ```XML
     <CODEC TYPE="X3" VERS="2">
   ```

5. List of parameters used by the coder.

   ```XML
     <BLKLEN>20</BLKLEN>
     <CODES N="4">RICE0,RICE1,RICE3,BFP</CODES>
     <FILTER>DIFF</FILTER>
     <NBITS>16</NBITS>
     <T N="3">3,8,20</T>
   ```

6. end the codec and cfg fields and append the metadata

   ```XML
     </CODEC></CFG>
   ```

Below is an example XML. The uncessary whitespace can be removed.

```XML
<X3ARCH PROG="x3new.m" VERSION="2.0" />
<CFG ID="0" FTYPE="XML" />
<CFG ID="1" FTYPE="WAV">
  <FS UNIT="Hz">16000</FS>
  <SUFFIX>wav</SUFFIX>
  <CODEC TYPE="X3" VERS="2">
    <BLKLEN>20</BLKLEN>
    <CODES N="4">RICE0,RICE1,RICE3,BFP</CODES>
    <FILTER>DIFF</FILTER>
    <NBITS>16</NBITS>
    <T N="3">3,8,20</T>
  </CODEC>
</CFG>
```

**`<Frame>`**

A frame contains content, it can either contain compressed audio content or other information. The
size of the frame is described by `<Frame Header>`.

```Text
Name            Type        Size (bytes)    Description

<Frame Header>  Bytes       20              Frame Header that describes the length of the compressed
                                            audio data.
<Filter State>  u16         2               The last audio sample from the previous frame. 0 if none.
<Block>         Bytes       Variable        The compressed audio data.
```

**`<Frame Header>`**

```Text
Name            Type        Size (bytes)    Description

<Frame Key>     String      2               A fixed value of "x3", or [0x78 0x33].  This can be used
                                            to discover the frame start if the compressed data is
                                            corrupted.
<Source Id>     u8          1               Data Source, as described in the "CFG" section in the XML
                                            document "ID".  Allows multiple data sources to be
                                            contained in one file.
<Num Channels>  u8          1               The number of channels contained in the frame.
<Num Samples>   u16         2               Samples in the frame: The number of uncompressed samples.
<Payload Length>u16         2               Bytes in the payload: The number of encoded bytes.
<Time>          u64         8               Time of first sample: The timestamp of the first sample in
                                            the frame.
<Header CRC>    u16         2               CRC of the frame header: The CRC of the above.
<Payload CRC>   u16         2               CRC of the payload: The CRC of the payload, all the blocks.
```

**`<Filter State>`**

```Text
Name            Type        Size (bytes)    Description

<Audio State>   u16         2               The last audio sample from the previous frame. 0 if none.
```

**`<Block>`**

This contains the x3 encoded audio data. See Encoding below.

NOTE: The block is word aligned. This means there will always be an even number of bytes in the
block.

## Encoding

Each block is encoded using the X3 protocol, each block may use a different encoder parameters than
the previous block. The audio data is filtered and the encoding is done on the filtered data.
Currently the only filter is the `diff` filter, where the change in amplitude of the audio data is
recorded.

There are two main ways of encoding data, these are:

- `BFP` or pass-through block: This keeps the filtered audio data in it's raw form, but removes
  unecessary leading bits.
- Rice code method: This uses lookup tables, called rice codes to get a good compression outcome.

## License

Implementation of the X3 lossless audio compression protocol.

Copyright (C) 2012,2013 Mark Johnson - Matlab Code
Copyright (C) 2019 Simon M. Werner - Rust Code

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,  
but WITHOUT ANY WARRANTY; without even the implied warranty of  
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the  
GNU General Public License for more details.

You should have received a copy of the GNU General Public License  
along with this program. If not, see <http://www.gnu.org/licenses/>.
