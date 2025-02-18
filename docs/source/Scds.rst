.. _scds-docs:

Scds
===========================

.. _Scds: https://github.com/kostkalab/scds
.. _publication: https://genomebiology.biomedcentral.com/articles/10.1186/s13059-024-03224-8

Scds_ is a transcription-based doublet detection software that uses two different methods to detect doublets - cxds and bcds.
The cxds method uses marker genes that are not co-expressed to identify droplets that are likely doublets.
bcds simulates doublet by adding droplet transcriptomes together and then uses variable genes to identify the probability a droplet is a doublet with a binary classification algorithm.
We typically use the combined score of these two methods but they can be use separately as well.
We have provided a wrapper script that takes common arguments for Scds_ and we also provide an example script that you can run manually in R if you prefer.



Data
----
This is the data that you will need to have prepare to run Scds_:

.. admonition:: Required
  :class: important

  - A counts matrix (``$COUNTS``)
  
    - The directory path containing your cellranger counts matrix files (directory containing ``barcodes.tsv``, ``genes.tsv`` and ``matrix.mtx`` **or** ``barcodes.tsv.gz``, ``features.tsv.gz`` and ``matrix.mtx.gz``)

      **or**

    - h5 file (``filtered_feature_bc_matrix.h5``) 

	  - If you don't have your data in this format, you can run Scds_ manually in R and load the data in using a method of your choosing.


.. admonition:: Optional

  - Output directory (``$SCDS_OUTDIR``)

    - If you don't provide an ``$SCDS_OUTDIR``, the results will be written to the present working directory.

  - Filtered barcode file

    - A list of barcodes that are a subset of the barcodes in your h5 or matrix.mtx files. This is useful if you have run other QC softwares such as `CellBender <https://cellbender.readthedocs.io/en/stable/index.html>`__ or `DropletQC <https://github.com/powellgenomicslab/DropletQC>`__ to remove empty droplets or droplets with damaged cells.

    - Expectation is that there is no header in this file
    

Run Scds
----------------
.. admonition:: :octicon:`stopwatch` Expected Resource Usage
  :class: note

  ~7min using a total of 10Gb memory when using 2 thread for the full :ref:`Test Dataset <TestData>` which contains ~20,982 droplets of 13 multiplexed donors,

You can either run Scds_ with the wrapper script we have provided or you can run it manually if you would prefer to alter more parameters.

.. tabs::

  .. tab:: With Wrapper Script

    First, let's assign the variables that will be used to execute each step.

    .. admonition:: Example Variable Settings
      :class: grey

      Below is an example of the variables that we can set up to be used in the command below.
      These are files provided as a :ref:`test dataset <TestData>` available in the :ref:`Data Preparation Documentation <DataPrep-docs>`
      Please replace paths with the full path to data on your system.

      .. code-block:: bash

        SCDS_OUTDIR=/path/to/output/scds
        COUNTS=/path/to/TestData4PipelineFull/test_dataset/outs/filtered_gene_bc_matrices/Homo_sapiens_GRCh38p10/


    To run Scds_ with our wrapper script, simply execute the following in your shell:

    .. code-block:: bash

		  singularity exec Demuxafy.sif scds.R -o $SCDS_OUTDIR -t $COUNTS

    .. admonition:: HELP! It says my file/directory doesn't exist!
      :class: dropdown

      If you receive an error indicating that a file or directory doesn't exist but you are sure that it does, this is likely an issue arising from Singularity.
      This is easy to fix.
      The issue and solution are explained in detail in the :ref:`Notes About Singularity Images <Singularity-docs>`

    To see all the parameters that this wrapper script will accept, run:

    .. code-block:: bash

      singularity exec Demuxafy.sif scds.R -h

        usage: scds.R
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

    This section demonstrates how to run Scds_ manually in R.

    First, you will have to start R.
    We have built R and all the required software to run Scds_ into the singularity image so you can run it directly from the image.

    .. code-block:: bash

      singularity exec Demuxafy.sif R

    That will open R in your terminal.
    Next, you can load all the libraries and run Scds_.

    .. code-block:: R

      .libPaths("/usr/local/lib/R/site-library") ### This is required so that R uses the libraries loaded in the image and not any local libraries
      library(dplyr)
      library(tidyr)
      library(tidyverse)
      library(scds)
      library(Seurat)
      library(SingleCellExperiment)

      ## Set up variables and parameters ##
      out <- "/path/to/scds/outdir/"
      tenX_matrix <- "/path/to/counts/matrix/dir/"

      ## Read in data
      counts <- Read10X(as.character(tenX_matrix), gene.column = 1) ## or Read10X_h5 if using h5 file as input

      ## Account for possibility that not just single cell data
      if (is.list(counts)){
        sce <- SingleCellExperiment(list(counts=counts[[grep("Gene", names(counts))]]))
      } else {
        sce <- SingleCellExperiment(list(counts=counts))
      }

      ## Annotate doublet using binary classification based doublet scoring:
      sce = bcds(sce, retRes = TRUE, estNdbl=TRUE)

      ## Annotate doublet using co-expression based doublet scoring:
      try({
          sce = cxds(sce, retRes = TRUE, estNdbl=TRUE)
      })

      ### If cxds worked, run hybrid, otherwise use bcds annotations
      if ("cxds_score" %in% colnames(colData(sce))) {
          ## Combine both annotations into a hybrid annotation
          sce = cxds_bcds_hybrid(sce, estNdbl=TRUE)
          Doublets <- as.data.frame(cbind(rownames(colData(sce)), colData(sce)$hybrid_score, colData(sce)$hybrid_call))
      } else {
          print("this pool failed cxds so results are just the bcds calls")
          Doublets <- as.data.frame(cbind(rownames(colData(sce)), colData(sce)$bcds_score, colData(sce)$bcds_call))
      }

      ## Doublet scores are now available via colData:
      colnames(Doublets) <- c("Barcode","scds_score","scds_DropletType")
      Doublets$scds_DropletType <- gsub("FALSE","singlet",Doublets$scds_DropletType) 
      Doublets$scds_DropletType <- gsub("TRUE","doublet",Doublets$scds_DropletType)

      message("writing output")
      write_delim(Doublets, paste0(out,"/scds_doublets_singlets.tsv"), "\t")


      summary <- as.data.frame(table(Doublets$scds_DropletType))
      colnames(summary) <- c("Classification", "Droplet N")
      write_delim(summary, paste0(out,"/scds_doublet_summary.tsv"), "\t")



  .. tab:: Run in R with Filtered Barcodes

    This section demonstrates how to run Scds_ manually in R and includes code to help filter for a subset of barcodes that are in the single cell data.
    
    First, you will have to start R.
    We have built R and all the required software to run Scds_ into the singularity image so you can run it directly from the image.

    .. code-block:: bash

      singularity exec Demuxafy.sif R

    That will open R in your terminal.
    Next, you can load all the libraries and run Scds_.

    .. code-block:: R

      .libPaths("/usr/local/lib/R/site-library") ### This is required so that R uses the libraries loaded in the image and not any local libraries
      library(dplyr)
      library(tidyr)
      library(tidyverse)
      library(scds)
      library(Seurat)
      library(SingleCellExperiment)

      ## Set up variables and parameters ##
      out <- "/path/to/scds/outdir/"
      tenX_matrix <- "/path/to/counts/matrix/dir/"
      filtered_barcodes_file <- "/path/to/counts/filtered/barcodes/file.tsv" ## can also be gzipped

      ## Read in data
      counts <- Read10X(as.character(tenX_matrix), gene.column = 1) ## or Read10X_h5 if using h5 file as input

      ## Read in filtered barcodes file
      filtered_barcodes <- read_delim(filtered_barcodes_file, delim = "\t", col_names = "Barcodes")

      ## Filter for the barcodes of interest
      ## Account for possibility that not just single cell data
        if (is.list(counts)){
            barcodes_head <- head(colnames(counts[[grep("Gene", names(counts))]]))
            counts <- counts[[grep("Gene", names(counts))]][, colnames(counts[[grep("Gene", names(counts))]]) %in% filtered_barcodes$Barcodes]
        } else {
            barcodes_head <- head(colnames(counts))
            counts <- counts[, colnames(counts) %in% filtered_barcodes$Barcodes]
        }


      ## Account for possibility that not just single cell data
      if (is.list(counts)){
        sce <- SingleCellExperiment(list(counts=counts[[grep("Gene", names(counts))]]))
      } else {
        sce <- SingleCellExperiment(list(counts=counts))
      }

      ## Annotate doublet using binary classification based doublet scoring:
      sce = bcds(sce, retRes = TRUE, estNdbl=TRUE)

      ## Annotate doublet using co-expression based doublet scoring:
      try({
          sce = cxds(sce, retRes = TRUE, estNdbl=TRUE)
      })

      ### If cxds worked, run hybrid, otherwise use bcds annotations
      if ("cxds_score" %in% colnames(colData(sce))) {
          ## Combine both annotations into a hybrid annotation
          sce = cxds_bcds_hybrid(sce, estNdbl=TRUE)
          Doublets <- as.data.frame(cbind(rownames(colData(sce)), colData(sce)$hybrid_score, colData(sce)$hybrid_call))
      } else {
          print("this pool failed cxds so results are just the bcds calls")
          Doublets <- as.data.frame(cbind(rownames(colData(sce)), colData(sce)$bcds_score, colData(sce)$bcds_call))
      }

      ## Doublet scores are now available via colData:
      colnames(Doublets) <- c("Barcode","scds_score","scds_DropletType")
      Doublets$scds_DropletType <- gsub("FALSE","singlet",Doublets$scds_DropletType) 
      Doublets$scds_DropletType <- gsub("TRUE","doublet",Doublets$scds_DropletType)

      message("writing output")
      write_delim(Doublets, paste0(out,"/scds_doublets_singlets.tsv"), "\t")


      summary <- as.data.frame(table(Doublets$scds_DropletType))
      colnames(summary) <- c("Classification", "Droplet N")
      write_delim(summary, paste0(out,"/scds_doublet_summary.tsv"), "\t")


Scds Results and Interpretation
----------------------------------------
After running the Scds_ with the wrapper script or manually you should have two files in the ``$SCDS_OUTDIR``:

.. code-block:: bash

	/path/to/output/scds
	├── scds_doublets_singlets.tsv
	└── scds_doublet_summary.tsv

- ``scds_doublet_summary.tsv``

  - A summary of the number of singlets and doublets predicted by Scds_.

    +----------------+-----------+
    |Classification  | Droplet N |
    +================+===========+
    |doublet         | 2771      |
    +----------------+-----------+
    |singlet         | 18211     |
    +----------------+-----------+

    - To check whether the number of doublets identified by Scds_ is consistent with the expected doublet rate expected based on the number of droplets that you captured, you can use our `Expected Doublet Estimation Calculator <test.html>`__.

- ``scds_doublets_singlets.tsv``

  - The per-barcode singlet and doublet classification from Scds_.
  
    +-------------------------+-------------------------+------------------+
    | Barcode                 | scds_score              | scds_DropletType |
    +=========================+=========================+==================+
    | AAACCTGAGATAGCAT-1      | 0.116344358493288       | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGCAGCGTA-1      | 0.539856378453988       | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGCGATGAC-1      | 0.0237184380134577      | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGCGTAGTG-1      | 0.163695865366576       | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGGAGTTTA-1      | 0.11591462421927        | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGGCTCATT-1      | 0.0479944175570073      | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGGGCACTA-1      | 0.374426050641161       | singlet          |
    +-------------------------+-------------------------+------------------+
    | AAACCTGAGTAATCCC-1      | 0.247842972104563       | singlet          |
    +-------------------------+-------------------------+------------------+
    | ...                     | ...                     | ...              |
    +-------------------------+-------------------------+------------------+


Merging Results with Other Software Retults
--------------------------------------------
We have provided a script that will help merge and summarize the results from multiple softwares together.
See :ref:`Combine Results <Combine-docs>`.

Citation
--------
If you used the Demuxafy platform for analysis, please reference our publication_ as well as `scds <https://academic.oup.com/bioinformatics/article/36/4/1150/5566507>`__.