.. # define a hard line break for HTML
.. |br| raw:: html

   <br />


.. _Vireo-docs:

Vireo
===========================
 
.. _Vireo: https://vireosnp.readthedocs.io/en/latest/manual.html
.. _publication: https://genomebiology.biomedcentral.com/articles/10.1186/s13059-024-03224-8

Vireo is a flexible demultiplexing software that can demutliplex without any reference SNP genotypes, with reference SNP genotypes for a subset of the donors in the pool or no reference SNP genotypes.
If you have reference SNP genotypes for **all** of the donors in your pool, you could also use :ref:`Demuxlet <Demuxlet-docs>` or :ref:`Souporcell <Souporcell-docs>`.
If you don't have reference SNP genotypes, you could alternatively use :ref:`Freemuxlet<Freemuxlet-docs>` or :ref:`ScSplit<scSplit-docs>`.




Data
----
This is the data that you will need to have preparede to run Vireo_:


.. admonition:: Required
  :class: important

  - Common SNP genotypes vcf (``$VCF``)

    - If you have reference SNP genotypes for individuals in your pool, you can use those

      - For Vireo_ you should only have the donors that are in this pool in the vcf file

    - If you do not have reference SNP genotypes, they can be from any large population resource (i.e. 1000 Genomes or HRC)

    - Filter for common SNPs (> 5% minor allele frequency) and SNPs overlapping genes

  - Barcode file (``$BARCODES``)

  - Number of samples in pool (``$N``)
  
  - Bam file (``$BAM``)

    - Aligned single cell reads

  - Output directory (``$VIREO_OUTDIR``)
  

.. admonition:: Optional

    - The SAM tag used in the Bam file to annotate the aligned single cell reads with their corresponding cell barcode (``$CELL_TAG``)

      - If not specified, _Vireo defaults to using ``CB``.

    - The SAM tag used in the Bam file to annotate the aligned single cell reads with their corresponding unique molecular identifier (UMI) (``$UMI_TAG``)

      - If not specified, _Vireo defaults to using ``UR``.
..
  Note: The default switched from ``UR`` to ``UB`` with cellsnp-lite v.1.2.3.
  The Demuxafy Singularity image v. 2.0.1 bundles cellsnp-lite v.1.2.1, so the
  old default applies. The above documentation should be updated to reflect the
  upstream change, once cellsnp-lite is updated to v.1.2.3 or higher in the
  Demuxafy Singularity image.




Run Vireo
------------
First, let's assign the variables that will be used to execute each step.

.. admonition:: Example Variable Settings
  :class: todo

    Below is an example of the variables that we can set up to be used in the command below.
    These are files provided as a :ref:`test dataset <TestData>` available in the :ref:`Data Preparation Documentation <DataPrep-docs>`
    Please replace paths with the full path to data on your system.

    .. code-block:: bash

      BARCODES=/path/to/TestData4PipelineFull/test_dataset/outs/filtered_gene_bc_matrices/Homo_sapiens_GRCh38p10/barcodes.tsv
      BAM=/path/to/test_dataset/possorted_genome_bam.bam
      VIREO_OUTDIR=/path/to/output/vireo
      VCF=/path/to/TestData4PipelineFull/test_dataset.vcf 
      N=14 ## optional


CellSNP Pileup
^^^^^^^^^^^^^^
.. admonition:: |:stopwatch:| Expected Resource Usage
  :class: note

  ~5h using a total of 5Gb memory when using 1 thread1 for the full :ref:`Test Dataset <TestData>` which contains ~20,982 droplets of 13 multiplexed donors,

First, you need to count the number of alleles at each SNP in each droplet using cellSNP-lite:

Please note that the ``\`` at the end of each line is purely for readability to put a separate parameter argument on each line.

.. code-block:: bash

  singularity exec Demuxafy.sif cellsnp_pileup.py \
            -s $BAM \
            -b $BARCODES \
            -O $VIREO_OUTDIR \
            -R $VCF \
            -p 20 \
            --minMAF 0.1 \
            --minCOUNT 20 \
            --cellTAG $CELL_TAG \
            --UMItag $UMI_TAG \
            --gzip 


You can alter the ``-p``, ``--minMAF`` and ``--minCOUNT`` parameters to fit your data and your needs.
We have found these settings to work well with our data.

.. admonition:: HELP! It says my file/directory doesn't exist!
  :class: dropdown

  If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
  This is easy to fix.
  The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`


If the pileup is successful, you will have this new file in your ``$VIREO_OUTDIR``:

.. code-block:: bash

	/path/to/output/vireo
	├── cellSNP.base.vcf.gz
	├── cellSNP.samples.tsv
	├── cellSNP.tag.AD.mtx
	├── cellSNP.tag.DP.mtx
	└── cellSNP.tag.OTH.mtx

Additional details about outputs are available below in the :ref:`Vireo Results and Interpretation <vireo-results>`.



Demultiplex with Vireo
^^^^^^^^^^^^^^^^^^^^^^
.. admonition:: |:stopwatch:| Expected Resource Usage
  :class: note

  ~2min using a <1Gb memory when using 2 threads for the full :ref:`Test Dataset <TestData>` which contains ~20,982 droplets of 13 multiplexed donors,

Next, we can use the cellSNP results to demultiplex the data with Vireo_.
As already mentioned, you can use Vireo_ with multiple different levels of reference SNP genotypes.
We've provided an example command for each of these differing amounts of donor SNP genotype data.

.. tabs::

  .. tab:: With SNP Genotype |br| Data for All Donors

    You will need to provide which genotype measure  (``$FIELD``) is provided in your donor SNP genotype file (GT, GP, or PL); default is PL.

    .. admonition:: STRONGLY Recommended
      :class: important

      For Vireo_ you should only have the donors that are in this pool in the vcf file.
      Vireo_ assumes all the individuals in your vcf are in the pool - so if left unfiltered, it will check for all the individuals in the reference SNP genotype file.

      Vireo_ also runs more efficiently when the SNPs from the donor ``$VCF`` have been filtered for the SNPs identified by ``cellSNP-lite``.
      Therefore, it is highly recommended subset the vcf first.

      We can do both of these filtering actions at the same time with `bcftools`:

        **Note:** If your reference SNP genotype ``$VCF`` is bgzipped (`i.e.` ends in ``.vcf.gz``), you should first bgzip and index your file with:

          .. code-block::

            singularity exec Demuxafy.sif bgzip -c $VCF > $VCF.gz
            singularity exec Demuxafy.sif tabix -p vcf $VCF.gz

        .. code-block::

          singularity exec Demuxafy.sif bcftools view $VCF \
                  -R $VIREO_OUTDIR/cellSNP.base.vcf.gz \
                  -s sample1,sample2 \
                  -Ov \
                  -o $VIREO_OUTDIR/donor_subset.vcf

        Alternatively, if you have the individuals from the pool in a file with each individuals separated by a new line (``individual_file.tsv``), then you can use ``-S individual_file.tsv``.


    To run Vireo_ with reference SNP genotype data for your donors (ideally filtered as shown above):

    Please note that the ``\`` at the end of each line is purely for readability to put a separate parameter argument on each line.

    .. code-block::

      singularity exec Demuxafy.sif vireo \
              -c $VIREO_OUTDIR \
              -d $VIREO_OUTDIR/donor_subset.vcf \
              -o $VIREO_OUTDIR \
              -t $FIELD \
              --callAmbientRNAs

    .. admonition:: HELP! It says my file/directory doesn't exist!
      :class: dropdown

      If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
      This is easy to fix.
      The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`


  .. tab:: With SNP Genotype |br| Data for Some Donors

    .. admonition:: STRONGLY Recommended

      For Vireo_ you should only have the donors that are in this pool in the reference SNP genotype vcf file. 
      Vireo assumes all the individuals in your vcf are in the pool - so if left unfiltered, it will check for all the individuals in the reference SNP genotype file.
      It assumes that ``$N`` is larger than the number of donors in the ``$VCF``

      Vireo_ also runs more efficiently when the SNPs from the donor ``$VCF`` have been filtered for the SNPs identified by ``cellSNP-lite``.
      Therefore, it is highly recommended to subset the vcf first.

      We can do both of these filtering actions at the same time with `bcftools`:

        **Note:** If your reference SNP genotype ``$VCF`` is bgzipped (`i.e.` ends in ``.vcf.gz``), you should first bgzip and index your file with:

          .. code-block::

            singularity exec Demuxafy.sif bgzip -c $VCF > $VCF.gz
            singularity exec Demuxafy.sif tabix -p vcf $VCF.gz

        .. code-block::

          singularity exec Demuxafy.sif bcftools view $VCF \
                  -R $VIREO_OUTDIR/cellSNP.base.vcf.gz \
                  -s sample1,sample2 \
                  -Ov \
                  -o $VIREO_OUTDIR/donor_subset.vcf

        Alternatively, if you have the individuals from the pool in a file with each individuals separated by a new line (``individual_file.tsv``), then you can use ``-S individual_file.tsv``.

    .. admonition:: Recommended
      :class: important

      Vireo runs more efficiently when the SNPs from the donor ``$VCF`` have been filtered for the SNPs identified by ``cellSNP-lite``.
      Therefore, it is highly recommended subset the vcf as follows first:

        .. code-block::

          singularity exec Demuxafy.sif bcftools view $VCF \
                  -R $VIREO_OUTDIR/cellSNP.base.vcf.gz \
                  -Oz \
                  -o $VIREO_OUTDIR/donor_subset.vcf

    Please note that the ``\`` at the end of each line is purely for readability to put a separate parameter argument on each line.

    .. code-block::

      singularity exec Demuxafy.sif vireo \
                -c $VIREO_OUTDIR \
                -d $VIREO_OUTDIR/donor_subset.vcf.gz \
                -o $VIREO_OUTDIR \
                -t $FIELD \
                -N $N \
                --callAmbientRNAs

    .. admonition:: HELP! It says my file/directory doesn't exist!
      :class: dropdown

      If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
      This is easy to fix.
      The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`

  .. tab:: Without Donor SNP |br| Genotype Data

    Please note that the ``\`` at the end of each line is purely for readability to put a separate parameter argument on each line.

    .. code-block::

      singularity exec Demuxafy.sif vireo \
              -c $VIREO_OUTDIR \
              -o $VIREO_OUTDIR \
              -N $N \
              --callAmbientRNAs

    .. admonition:: HELP! It says my file/directory doesn't exist!
      :class: dropdown

      If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
      This is easy to fix.
      The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`

If Vireo_ is successful, you will have these new files in your ``$VIREO_OUTDIR``:

.. code-block:: bash
  :emphasize-lines: 7,8,9,10,11,12,13

  /path/to/output/vireo
  ├── cellSNP.base.vcf
  ├── cellSNP.samples.tsv
  ├── cellSNP.tag.AD.mtx
  ├── cellSNP.tag.DP.mtx
  ├── cellSNP.tag.OTH.mtx
  ├── donor_ids.tsv
  ├── donor_subset.vcf
  ├── fig_GT_distance_estimated.pdf
  ├── _log.txt
  ├── prob_doublet.tsv.gz
  ├── prob_singlet.tsv.gz
  └── summary.tsv

Additional details about outputs are available below in the :ref:`Vireo Results and Interpretation <vireo-results>`.


.. _vireo-results:

Vireo Results and Interpretation
-------------------------------------
After running the Vireo_ steps, you will have a number of files in your ``$VIREO_OUTDIR``. 
These are the files that most users will find the most informative:

- ``summary.tsv``

  - A summary of the droplets assigned to each donor, doublets and unassigned.

    +------------+------+
    | Var1       | Freq |
    +============+======+
    | 113_113    | 1342 |
    +------------+------+
    | 349_350    | 1475 |
    +------------+------+
    | 352_353    | 1619 |
    +------------+------+
    | 39_39      | 1309 |
    +------------+------+
    | 40_40      | 1097 |
    +------------+------+
    | 41_41      | 1144 |
    +------------+------+
    | 42_42      | 1430 |
    +------------+------+
    | 43_43      | 1561 |
    +------------+------+
    | 465_466    | 1104 |
    +------------+------+
    | 596_597    | 1271 |
    +------------+------+
    | 597_598    | 1532 |
    +------------+------+
    | 632_633    | 871  |
    +------------+------+
    | 633_634    | 967  |
    +------------+------+
    | 660_661    | 1377 |
    +------------+------+
    | doublet    | 2770 |
    +------------+------+
    | unassigned | 113  |
    +------------+------+

    - To check whether the number of doublets identified by Vireo_ is consistent with the expected doublet rate based on the number of droplets that you captured, you can use our `Expected Doublet Estimation Calculator <test.html>`__.


- ``donor_ids.tsv``

  - The classification of each droplet, and some droplet metrics.

    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | cell                    | donor_id|        prob_max | prob_doublet    | n_vars  | best_singlet |  best_doublet    |
    +=========================+=========+=================+=================+=========+==============+==================+
    | AAACCTGAGATAGCAT-1      | 41_41   | 1.00e+00        | 9.13e-09        | 115     | 41_41        | 40_40,41_41      |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | AAACCTGAGCAGCGTA-1      | 465_466 | 1.00e+00        | 5.03e-17        | 239     | 465_466      | 349_350,43_43    |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | AAACCTGAGCGATGAC-1      | 113_113 | 1.00e+00        | 7.57e-07        | 98      | 113_113      | 113_113,633_634  |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | AAACCTGAGCGTAGTG-1      | 349_350 | 1.00e+00        | 8.07e-07        | 140     | 349_350      | 349_350,597_598  |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | AAACCTGAGGAGTTTA-1      | 632_633 | 1.00e+00        | 5.99e-11        | 177     | 632_633      | 40_40,113_113    |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+
    | AAACCTGAGGCTCATT-1      | 39_39   | 1.00e+00        | 4.44e-06        | 110     | 39_39        | 39_39,40_40      |
    +-------------------------+---------+-----------------+-----------------+---------+--------------+------------------+


Merging Results with Other Software Results
--------------------------------------------
We have provided a script that will help merge and summarize the results from multiple softwares together.
See :ref:`Combine Results <Combine-docs>`.

Citation
--------
If you used the Demuxafy platform for analysis, please reference our publication_ as well as `Vireo <https://genomebiology.biomedcentral.com/articles/10.1186/s13059-019-1865-2>`__.
