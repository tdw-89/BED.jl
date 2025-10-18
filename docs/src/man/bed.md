# BED
BED is a text-based file format for representing genomic annotations like genes, transcripts, and so on.
A BED file has tab-delimited and variable-length fields; the first three fields denoting a genomic interval are mandatory.

This is an example of RNA transcripts:
```
chr9	68331023	68424451	NM_015110	0	+
chr9	68456943	68486659	NM_001206	0	-
```

The `BED` package supports I/O for BED by providing the following three types:
* Reader type: `BED.Reader`
* Writer type: `BED.Writer`
* Element type: `BED.Record`

## Examples

Here is a common workflow to iterate over all records in a BED file:
```julia
# Import the BED module.
using BED

# Open a BED file.
reader = open(BED.Reader, "data.bed")

# Iterate over records.
for record in reader
    # Do something on record (see Accessors section).
    chrom = BED.chrom(record)
    # ...
end

# Finally, close the reader.
close(reader)
```

The iterator interface demonstrated above allocates an object for each record and that may be a bottleneck of reading data from a file.
In-place reading reuses a pre-allocated object for every record and less memory allocation happens in reading:

```julia
# Import the BED module.
using BED

# Open a BED file.
reader = open(BED.Reader, "data.bed")

# Pre-allocate record.
record = BED.Record()
while !eof(reader)
    empty!(record)
    read!(reader, record)
    # do something
end

# Finally, close the reader.
close(reader)
```

If you repeatedly access records within specific ranges, it would be more efficient to construct an `IntervalCollection` object from a BED reader:
```julia
using BED
using GenomicFeatures

# Create an interval collection in memory.
icol = open(BED.Reader, "data.bed") do reader
    IntervalCollection(reader)
end

# Query overlapping records.
for interval in eachoverlap(icol, Interval("chrX", 40001, 51500))
    # A record is stored in the metadata field of an interval.
    record = metadata(interval)
    # ...
end
```

## narrowPeak Format Support

The BED package also supports the narrowPeak format, which is a specialized BED format used by peak-calling tools like MACS3. narrowPeak files contain 10 columns with the following structure:

1. `chrom` - Chromosome name
2. `chromStart` - Start position (0-based)
3. `chromEnd` - End position (not included)
4. `name` - Peak name
5. `score` - Integer score (0-1000)
6. `strand` - Strand (+, -, or .)
7. `signalValue` - Measurement of overall enrichment (floating-point)
8. `pValue` - Statistical significance as -log10(pValue) (floating-point)
9. `qValue` - Statistical significance as -log10(qValue) using FDR (floating-point)
10. `peak` - Point-source called for this peak; 0-based offset from chromStart (integer)

narrowPeak files are automatically handled by the BED reader and provide additional accessor functions:

```julia
using BED

# Open a narrowPeak file
reader = open(BED.Reader, "peaks.narrowPeak")

for record in reader
    # Access standard BED fields
    chrom = BED.chrom(record)
    start = BED.chromstart(record)
    peak_name = BED.name(record)
    
    # Access narrowPeak-specific fields
    signal = BED.signalvalue(record)  # Overall enrichment
    pval = BED.pvalue(record)         # -log10(p-value)
    qval = BED.qvalue(record)         # -log10(q-value)
    peak_offset = BED.peak(record)    # Offset from chromStart
    
    # Calculate absolute peak position
    peak_pos = start + peak_offset
end

close(reader)
```
