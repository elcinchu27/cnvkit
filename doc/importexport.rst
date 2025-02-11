Compatibility and other I/O
===========================


.. _version:

``version``
-----------

Print CNVkit's version as a string on standard output::

    cnvkit.py version

If you submit a bug report or feature request for CNVkit, please include the
CNVkit version in your message so we can help you more efficiently.


.. _import-picard:

``import-picard``
-----------------

Convert Picard CalculateHsMetrics per-target coverage files (.csv) to the
CNVkit .cnn format::

    cnvkit.py import-picard *.hsmetrics.targetcoverages.csv *.hsmetrics.antitargetcoverages.csv
    cnvkit.py import-picard picard-hsmetrics/ -d cnvkit-from-picard/

You can use `Picard tools <http://broadinstitute.github.io/picard/>`_ to perform
the bin read depth and GC calculations that CNVkit normally performs with the
:ref:`coverage` and :ref:`reference` commands, if need be.

Procedure:

1. Use the :ref:`target` and :ref:`antitarget` commands to generate the
   "targets.bed" and "antitargets.bed" files.
2. Convert those BED files to Picard's "interval list" format by adding the BAM
   header to the top of the BED file and rearranging the columns -- see the
   Picard command `BedToIntervalList
   <http://broadinstitute.github.io/picard/command-line-overview.html#BedToIntervalList>`_.
3. Run Picard `CalculateHsMetrics
   <http://broadinstitute.github.io/picard/command-line-overview.html#CalculateHsMetrics>`_
   on each of your normal/control BAM files with the "targets" and "antitargets"
   interval lists (separately), your reference genome, and the
   "PER_TARGET_COVERAGE" option.
4. Use :ref:`import-picard` to convert all of the PER_TARGET_COVERAGE files to
   CNVkit's .cnn format.
5. Use :ref:`reference` to build a CNVkit reference from those .cnn files. It
   will retain the GC values Picard calculated; you don't need to provide the
   reference genome sequence again to get GC (but you if you do, it will also
   calculate the RepeatMaster fraction values)
6. Use :ref:`batch` with the ``-r``/``--reference`` option to process the rest
   of your test samples.


.. _import-seg:

``import-seg``
--------------

Convert a file in the :ref:`segformat` format (e.g. the output of standard CBS
or the GenePattern server) into one or more CNVkit .cns files.

The chromosomes in a SEG file may have been converted from chromosome names to
integer IDs. Options in ``import-seg`` can help recover the original names.

* To add a "chr" prefix, use "-p chr".
* To convert chromosome indices 23, 24 and 25 to the names "X", "Y" and "M" (a
  common convention), use "-c human".
* To use an arbitrary mapping of indices to chromosome names, use a
  comma-separated "key:value" string. For example, the human convention would
  be: "-c 23:X,24:Y,25:M".


.. _import-theta:

``import-theta``
----------------

Convert the ".results" output of `THetA2
<http://compbio.cs.brown.edu/projects/theta/>`_ to one or more CNVkit .cns files
representing subclones with integer absolute copy number in each segment.

::

    cnvkit.py import-theta Sample.cns Sample.BEST.results

See the page on tumor :doc:`heterogeneity` for more guidance on performing this
analysis.

.. _export:

``export``
----------

Convert copy number ratio tables (.cnr files) or segments (.cns) to
another format.

bed
```

Segments can be exported to BED format to support a variety of other uses, such
as viewing in a genome browser.
By default only regions with copy number different from the given ploidy
(default 2) are output. (Notice what this means for allosomes.)
To output all segments, use the ``--show all`` option.

The BED format represents integer copy numbers in absolute scale, not log2
ratios.  If the input .cns file contains a "cn" column with integer copy number
values, as generated by the :ref:`call` command, `export bed` will use those
values. Otherwise the log2 ratio value of each input segment is converted and
rounded to an integer value, similar to the `call -m clonal` method.

::

    # Estimate integer copy number of each segment
    cnvkit.py call Sample.cns -y -o Sample.call.cns
    # Show estimated integer copy number of all regions
    cnvkit.py export bed Sample.call.cns --show all -y -o Sample.bed

The same BED format can also specify CNV regions to the FreeBayes variant caller
with FreeBayes's ``--cnv-map`` option::

    # Show only CNV regions
    cnvkit.py export bed Sample.call.cns -o all-samples.cnv-map.bed

vcf
```

Convert segments, ideally already adjusted by the :ref:`call` command, to a
:ref:`vcfformat` file. Copy ratios are converted to absolute integers, as with
BED export, and VCF records are created for the segments where the copy number
is different from the expected ploidy (e.g. 2 on autosomes, 1 on haploid sex
chromosomes, depending on sample sex).

A sample's :doc:`chromosomal sex <sex>` can be specified with the
``-x``/``--sample-sex`` option, or will be guessed automatically.
If the option ``-y`` / ``--male-reference`` / ``--haploid-x-reference`` was used
to construct the :ref:`reference`, use it here, too.

::

    cnvkit.py export vcf Sample.cns -y -g female -i "SampleID" -o Sample.cnv.vcf

cdt, jtv
````````

A collection of probe-level copy ratio files (``*.cnr``) can be exported to Java
TreeView via the standard CDT format or a plain text table::

    cnvkit.py export jtv *.cnr -o Samples-JTV.txt
    cnvkit.py export cdt *.cnr -o Samples.cdt

seg
```

Similarly, the segmentation files for multiple samples (``*.cns``) can be
exported to the standard SEG format to be loaded in the Integrative Genomic
Viewer (IGV)::

    cnvkit.py export seg *.cns -o Samples.seg


nexus-basic
```````````

The format ``nexus-basic`` can be loaded directly by the commercial program
Biodiscovery Nexus Copy Number, specifying the "basic" input format in that
program. This allows viewing CNVkit data as if it were from array CGH.

This is a tabular format very similar to .cnr files, with the columns:

#. chromosome
#. start
#. end
#. log2


nexus-ogt
`````````

The format ``nexus-ogt`` can be loaded directly by the commercial program
Biodiscovery Nexus Copy Number, specifying the "Custom-OGT" input format in that
program. This allows viewing CNVkit data as if it were from a SNP array.

This is a tabular format similar to .cnr files, but with B-allele frequencies
(BAFs) extracted from a corresponding VCF file. The format's columns are (with
.cnr equivalents):

#. "Chromosome" (chromosome)
#. "Position" (start)
#. "Position" (end)
#. "Log R Ratio" (log2)
#. "B-Allele Frequency" (from VCF)

The positions of each heterozygous variant record in the given VCF are matched
to bins in the given .cnr file, and the variant allele frequencies are extracted
and assigned to the matching bins.

- If a bin contains no variants, the BAF field is left blank
- If a bin contains multiple variants, the BAFs of those variants are "mirrored"
  to be all above .5 (e.g. BAF of .3 becomes .7), then the median is taken as
  the bin-wide BAF.


.. _export_theta:

theta
`````

`THetA2 <http://compbio.cs.brown.edu/projects/theta/>`_ is a program for
estimating normal-cell contamination and tumor subclone population fractions
based on a tumor sample's copy number profile and, optionally, SNP allele
frequencies. (See the page on tumor :doc:`heterogeneity` for more guidance.)

THetA2's input file is a BED-like file, typically with the extension
``.interval_count``, listing the read counts  within each copy-number segment in
a pair of tumor and normal samples.
CNVkit can generate this file given the CNVkit-inferred tumor segmentation
(.cns), bypassing the initial step of THetA2, CreateExomeInput, which counts the
reads in each sample's BAM file.

The normal-sample read counts in this file are used for weighting each segment
in THetA2's calculations. We recommend providing these to ``export theta`` via
the CNVkit pooled or paired reference file (.cnn) you created for your panel::

    # From an existing CNVkit reference
    cnvkit.py export theta Sample_Tumor.cns reference.cnn -o Sample.theta2.interval_count

The THetA2 normal read counts can also be derived from the normal sample's bin
log2 ratios, if for some reason this is all you have::

    # From a paired normal sample
    cnvkit.py export theta Sample_Tumor.cns Sample_Normal.cnr -o Sample.theta2.interval_count

If neither file is given, the THetA2 normal read counts will be calculated from
the segment weight values in the given .cns file, or the number of probes if the
"weight" column is missing, or as a last resort, the segment sizes if the
"probes" column is also missing::

    # From segment weights and/or probe counts
    cnvkit.py export theta Sample_Tumor.cns -o Sample.theta2.interval_count


THetA2 also can take the tumor and normal samples' SNP allele frequencies as
input to improve its estimates. THetA2 uses another custom format for these
values, and provides another script for creating these files from VCF that we'd
again prefer to bypass. CNVkit's ``export theta`` command produces these two
additional files when given a VCF file of paired tumor-normal SNV calls with the
``-v``/``--vcf`` option::

    cnvkit.py export theta Sample_Tumor.cns reference.cnn -v Sample_Paired.vcf

This produces three output files; ``-o`` will be used for the read count file,
while the SNV allele count files will be named according to the .cns file, e.g.
``Sample_Tumor.tumor.snp_formatted.txt`` and
``Sample_Tumor.normal.snp_formatted.txt``.
