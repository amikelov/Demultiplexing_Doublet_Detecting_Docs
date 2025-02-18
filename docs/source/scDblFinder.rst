.. _scDblFinder-docs:

ScDblFinder
===========================

.. _ScDblFinder: https://github.com/plger/scDblFinder
.. _publication: https://genomebiology.biomedcentral.com/articles/10.1186/s13059-024-03224-8


scDblFinder is a transcriptome-based doublet detecting method that uses doublet simulation from droplets in the dataset to identify doublets.
We have provided a wrapper script that takes common arguments for ScDblFinder_ and also provide example code for you to run manually if you prefer.



Data
----
This is the data that you will need to have prepare to run ScDblFinder_:

.. admonition:: Required
  :class: important

  - A counts matrix (``$COUNTS``)
  
    - The directory path containing your cellranger counts matrix files (directory containing ``barcodes.tsv``, ``genes.tsv`` and ``matrix.mtx`` **or** ``barcodes.tsv.gz``, ``features.tsv.gz`` and ``matrix.mtx.gz``)

      **or**

    - h5 file (``filtered_feature_bc_matrix.h5``) 

	  - If you don't have your data in this format, you can run ScDblFinder_ manually in R and load the data in using a method of your choosing.

.. admonition:: Optional

  - Output directory (``$SCDBLFINDER_OUTDIR``)

    - If you don't provide an ``$SCDBLFINDER_OUTDIR``, the results will be written to the present working directory.

  - Filtered barcode file

    - A list of barcodes that are a subset of the barcodes in your h5 or matrix.mtx files. This is useful if you have run other QC softwares such as `CellBender <https://cellbender.readthedocs.io/en/stable/index.html>`__ or `DropletQC <https://github.com/powellgenomicslab/DropletQC>`__ to remove empty droplets or droplets with damaged cells.

    - Expectation is that there is no header in this file



Run ScDblFinder
----------------
.. admonition:: :octicon:`stopwatch` Expected Resource Usage
  :class: note

  ~1min using a total of 3Gb memory when using 2 thread for the full :ref:`Test Dataset <TestData>` which contains ~20,982 droplets of 13 multiplexed donors,

You can either run ScDblFinder_ with the wrapper script we have provided or you can run it manually if you would prefer to alter more parameters.

.. tabs::

  .. tab:: With Wrapper Script
    First, let's assign the variables that will be used to execute each step.

    .. admonition:: Example Variable Settings
      :class: grey

        Below is an example of the variables that we can set up to be used in the command below.
        These are files provided as a :ref:`test dataset <TestData>` available in the :ref:`Data Preparation Documentation <DataPrep-docs>`
        Please replace paths with the full path to data on your system.

        .. code-block:: bash

          COUNTS=/path/to/TestData4PipelineFull/test_dataset/outs/filtered_gene_bc_matrices/Homo_sapiens_GRCh38p10/
          SCDBLFINDER_OUTDIR=/path/to/output/scDblFinder


    To run ScDblFinder_ with our wrapper script, simply execute the following in your shell:
    
    .. code-block:: bash

		  singularity exec Demuxafy.sif scDblFinder.R -o $SCDBLFINDER_OUTDIR -t $COUNTS

    .. admonition:: HELP! It says my file/directory doesn't exist!
      :class: dropdown

      If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
      This is easy to fix.
      The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`

    To see all the parameters that this wrapper script will accept, run:

    .. code-block:: bash

      singularity exec Demuxafy.sif scDblFinder.R -h

        usage: scDblFinder.R file.
              [-h] -o OUT -t TENX_MATRIX [-b BARCODES_FILTERED]
            
        optional arguments:
          -h, --help            show this help message and exit
          -o OUT, --out OUT     The output directory where results will be saved
          -t TENX_MATRIX, --tenX_matrix TENX_MATRIX
                                Path to the 10x filtered matrix directory or h5 file.
          -b BARCODES_FILTERED, --barcodes_filtered BARCODES_FILTERED
                                Path to a list of filtered barcodes to use for doublet
                                detection.


  .. tab:: Run in R

    This section demonstrates how to run ScDblFinder_ manually in R.

    First, you will have to start R.
    We have built R and all the required software to run ScDblFinder_ into the singularity image so you can run it directly from the image.

    .. code-block:: bash

      singularity exec Demuxafy.sif R


    That will open R in your terminal.
    Next, you can load all the libraries and run ScDblFinder_.

    .. code-block:: R

      .libPaths("/usr/local/lib/R/site-library") ### This is required so that R uses the libraries loaded in the image and not any local libraries
      library(scDblFinder)
      library(Seurat)
      library(SingleCellExperiment)
      library(tidyverse)

      ## Set up variables and parameters ##
      out <- "/path/to/scds/outdir/"
      tenX_matrix <- "/path/to/counts/matrix/dir/"

      dir.create(out, recursive = TRUE)
      print(paste0("Using the following counts directory: ", tenX_matrix))



      ### Read in data as an sce object ###
      counts <- Read10X(tenX_matrix, gene.column = 1) ## or Read10X_h5 if using h5 file as input
      sce <- SingleCellExperiment(list(counts=counts))


      ## Calculate doublet ratio ###
      doublet_ratio <- ncol(sce)/1000*0.008


      ### Calculate Singlets and Doublets ###
      sce <- scDblFinder(sce, dbr=doublet_ratio)


      
      ### Make a dataframe of the results ###
      results <- data.frame("Barcode" = rownames(colData(sce)), "scDblFinder_DropletType" = sce$scDblFinder.class, "scDblFinder_Score" = sce$scDblFinder.score)


      write_delim(results, path = paste0(out,"/scDblFinder_doublets_singlets.tsv"), delim = "\t")

      ### Calculate number of doublets and singlets ###
      summary <- as.data.frame(table(results$scDblFinder_DropletType))
      colnames(summary) <- c("Classification", "Droplet N")
      write_delim(summary, paste0(out,"/scDblFinder_doublet_summary.tsv"), "\t")


  .. tab:: Run in R with Filtered Barcodes

    This section demonstrates how to run ScDblFinder_ manually in R and includes code to help filter for a subset of barcodes that are in the single cell data.
    
    First, you will have to start R.
    We have built R and all the required software to run ScDblFinder_ into the singularity image so you can run it directly from the image.

    .. code-block:: bash

      singularity exec Demuxafy.sif R


    That will open R in your terminal.
    Next, you can load all the libraries and run ScDblFinder_.

    .. code-block:: R

      .libPaths("/usr/local/lib/R/site-library") ### This is required so that R uses the libraries loaded in the image and not any local libraries
      library(scDblFinder)
      library(Seurat)
      library(SingleCellExperiment)
      library(tidyverse)

      ## Set up variables and parameters ##
      out <- "/path/to/scds/outdir/"
      tenX_matrix <- "/path/to/counts/matrix/dir/"
      filtered_barcodes_file <- "/path/to/counts/filtered/barcodes/file.tsv" ## can also be gzipped

      dir.create(out, recursive = TRUE)
      print(paste0("Using the following counts directory: ", tenX_matrix))



      ### Read in data as an sce object and filtered barcodes ###
      counts <- Read10X(tenX_matrix, gene.column = 1) ## or Read10X_h5 if using h5 file as input
      filtered_barcodes <- read_delim(args$barcodes_filtered, delim = "\t", col_names = "Barcodes")

      ## Filter for the barcodes list of interest
      if (is.list(counts)){
          counts <- counts[[grep("Gene", names(counts))]][, colnames(counts[[grep("Gene", names(counts))]]) %in% filtered_barcodes$Barcodes]
      } else {
          counts <- counts[, colnames(counts) %in% filtered_barcodes$Barcodes]
      }

      ## Create ingle cell experiment object
      sce <- SingleCellExperiment(list(counts=counts))


      ## Calculate doublet ratio ###
      doublet_ratio <- ncol(sce)/1000*0.008


      ### Calculate Singlets and Doublets ###
      sce <- scDblFinder(sce, dbr=doublet_ratio)


      
      ### Make a dataframe of the results ###
      results <- data.frame("Barcode" = rownames(colData(sce)), "scDblFinder_DropletType" = sce$scDblFinder.class, "scDblFinder_Score" = sce$scDblFinder.score)


      write_delim(results, path = paste0(out,"/scDblFinder_doublets_singlets.tsv"), delim = "\t")

      ### Calculate number of doublets and singlets ###
      summary <- as.data.frame(table(results$scDblFinder_DropletType))
      colnames(summary) <- c("Classification", "Droplet N")
      write_delim(summary, paste0(out,"/scDblFinder_doublet_summary.tsv"), "\t")




ScDblFinder Results and Interpretation
----------------------------------------
After running the ScDblFinder_ with the wrapper script or manually you should have two files in the ``$SCDBLFINDER_OUTDIR``:

.. code-block:: bash

	/path/to/output/scDblFinder
	├── scDblFinder_doublets_singlets.tsv
	└── scDblFinder_doublet_summary.tsv

Here's a more detaild description of each of those files:

- ``scDblFinder_doublet_summary.tsv``

  - A sumamry of the number of singlets and doublets predicted by ScDblFinder_.

    +----------------+-----------+
    |Classification  | Droplet N |
    +================+===========+
    |doublet         | 3323      |
    +----------------+-----------+
    |singlet         | 17659     |
    +----------------+-----------+

    - To check whether the numbe of doublets identified by ScDblFinder_ is consistent with the expected doublet rate expected based on the number of droplets that you captured, you can use our `Expected Doublet Estimation Calculator <test.html>`__.

- ``scDblFinder_doublets_singlets.tsv``

  - The per-barcode singlet and doublet classification from ScDblFinder_.

    +-------------------------+-------------------------+--------------------------+
    | Barcode                 | scDblFinder_DropletType | scDblFinder_Score        |
    +=========================+=========================+==========================+
    | AAACCTGAGATAGCAT-1      | singlet                 | 0.0033526041079312563    |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGCAGCGTA-1      | doublet                 | 0.9937564134597778       |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGCGATGAC-1      | singlet                 | 5.045032594352961e-      |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGCGTAGTG-1      | singlet                 | 0.007504515815526247     |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGGAGTTTA-1      | singlet                 | 0.00835108570754528      |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGGCTCATT-1      | singlet                 | 0.028838597238063812     |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGGGCACTA-1      | doublet                 | 0.9985504746437073       |
    +-------------------------+-------------------------+--------------------------+
    | AAACCTGAGTAATCCC-1      | singlet                 | 0.005869860760867596     |
    +-------------------------+-------------------------+--------------------------+
    | ...                     | ...                     | ...                      |
    +-------------------------+-------------------------+--------------------------+


Merging Results with Other Software Results
--------------------------------------------
We have provided a script that will help merge and summarize the results from multiple softwares together.
See :ref:`Combine Results <Combine-docs>`.

Citation
--------
If you used the Demuxafy platform for analysis, please reference our publication_ as well as `ScDblFinder <https://github.com/plger/scDblFinder>`__.

