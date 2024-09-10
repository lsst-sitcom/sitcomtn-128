#############################################
Unrecognized Blends in Operations Rehearsal 3
#############################################

.. abstract::
   Unrecognized blends are a class of blended objects that are mistakenly identified as a single object.

       * We can identify such blends using higher resolution imaging from a space based telescope that will not be affected by seeing or using a truth catalog in the case of Operations Rehearsal 3.
       * We implement a simple matching algorithm using the RA and DEC of objects to label observed objects as isolated objects, recognized blends, or unrecognized blends.
       * We want to use commissioning to investigate the fraction of unrecognized blends and how different variables influence blends
       * We've created the pipeline to do that. Once space based data of the same region is processed in the pipeline we are good to go
       * Created with both object and source table in mind so will work on coadds as well (once we have those)
..   Using a space based catalog we can attempt to match objects between the two and identify any unrecognized blends. In this technote we use the truth catalogs as a proxy and create a simple matching algorithm between truth and observation to label recognized and unrecognized blends. We then investigate how the rate of unrecognized blends varies with object properties such as i-mag and local density.


Data
===============
   * Introduce Operations Rehearsal 3 with 3 nights of simulation
   * We are using the "Intermittent Cumulative DRP" catalog
   * We need to use the truth catalogs for matching
   * Truth catalogs use galaxies, stars, solar system objects and we are only looking at galaxies and stars
   * We apply the :code:`detect_isPrimary` flag which

        * Works with deblended children, removes sky objects, and is only inner regions
        * Might be a problem to apply cuts before matching but good for understandability
        * Double check!

   * In the observed catalog we only have :code:`extendedness` which is 1 for extended objects. We assume all extended objects are galaxies and use the two interchangibly 

The catalog can be accessed using the :code:`/repo/embargo` repo and found in the :code:`LSSTComCamSim/runs/intermittentcumulativeDRP/20240402_03_04/d_2024_03_29/DM-43865` collection.
The simulated images are built on top of a set of DC2 patches with input catalogs in :code:`/sdf/data/rubin/shared/ops-rehearsals/ops-rehearsal-3/imSim_catalogs`. 

Matching
========
   * We have to match between the space (truth) and ground catalogs to label unrecognized blends

        * Maybe include the nice egg recognized/unrecognized graphic? 

   * Easiest way to match objects is to use RA and DEC (after aligning astrometry) and taking the nearest match
   * Matching can go either direction but let's use the ground based RA and DEC.
   * This plays very well with isolated galaxies but is problematic for blends
   * Instead of taking the nearest match, we can query for objects in a radius and focus on the count

        * Querying around the ground RA and DEC in the ground catalog we have :math:`N_g` ground objects
        * Querying around the ground RA and DEC in the space catalog we have :math:`N_s` space objects
        * If :math:`N_s > N_g` we have a *candidate unrecognized blend*
        * If :math:`N_s \leq N_g` we have a **recognized blend** or a **spurious detection**.

   * Promoting a *candidate unrecognized blend* requires the truth objects to pass 2 cuts:

        * :math:`\Delta i < 2.5`
        * :math:`m_b > 28`

   * We use kd-trees since they are good and fast at this sort of stuff
   * Can do this on source (single visit) and object table
   * Other options are available! Ellipse overlap + :code:`friendly` that gives *blend entropy*
   * :code:`friendly` is being integrated into the pipeline and results on DC2 (not directly on OR3) are shown below

.. note::
   I removed a section on Recognized Blends that motivates the use for 1 arcsecond matching but not super rigorously.

.. 
        Recognized Blends
        ===================
        As mentioned above, matching with RA/DEC is fast using the k-d tree but applying magnitude cuts and magnitude difference cuts can be slow.
        It is worthwhile to reduce the number of candidates which motivates choosing a :math:`r` that will avoid most recognized blends.
        We can look at the distance between objects in recognized blends and choose a radius that rejects most of these blends.
        We use a distance of 1'' as :math:`> 99\%` of recognized blends are larger while also allowing for any issues with astrometry or centroid algorithms.

        .. image:: ./_static/recognized_blend_dist.png

        Figure 1. Distribution of distance between deblended children in the same parent. 


        .. We have no further use for recognized blends but it is possible to assign each detected ground object a :code:`primary-match` that then allows for direct comaprison against the space measurements and getting the error in galaxy photometry, shape measurements, and photo-z.


Unrecognized Blends
==============================

Using the kd-tree matching algorithm we can label unrecognized blends in the ground catalog. 
We then investigate how unrecognized blends correlate with various parameters such as object i-magnitude, shape parameters, and local density.

Magnitude Dependence
--------------------------
The fraction of unrecognized blends as a function of the observed *i-mag* is shown below including the results using :code:`friendly` on DC2.

.. image:: _static/unrec_blend_dc2_comparison.png

Figure 2.  Fraction of unrecognized blends as a function of observed *i-mag*. The kd results are shown in blue and results using blend entropy in orange. The two show a similar bump at the faint end while there is not strict agreement.

If we restrict to only detected galaxies we se a slight increase

.. image:: ./_static/unrec_blend_extended.png

Figure 3. Fraction of unrecognized blend as a function of observed i-mag including a restriction on only observed galaxies. There is a slight increase corresponding to extended objects being easier to overlap with.

Shape Parameters
-----------------
Given that there is little to no difference among the shape parameters, this gives good confidence that the pixel grid is not impacting shape measurements and unrecognized blends in strange ways

We look at the second moment, :math:`Q_{ij}`, of extended objects.
We combine the second moments via 

.. math::
   e_1 = \frac{Q_{xx} - Q_{yy}}{Q_{xx} + Q_{yy}} \;\;\; e_2 = \frac{2Q_{xy}}{Q_{xx} + Q_{yy}}

We create :math:`Q_{rr} = \sqrt{Q_{xx}^2 + Q_{yy}^2}`.

.. image:: ./_static/unrec_blend_shapeij.png

Figure 4. Fraction of unrecognized blend as a function of measured second moments on observed galaxies. The range is limited to the 95% range for each measurement.

.. image:: ./_static/unrec_blend_pol.png

Figure 5. Fraction of unrecognized blend as a function of ellipse polarization on observed galaxies.



Local Density
--------------

To estimate the local density, :math:`\sum(r_i)`, we use Equation 7 from `Darvish et al <https://arxiv.org/pdf/1503.07879.pdf>`_.

.. math::
   \sum(r_i) = \frac{\sum_{j=1}^k j}{\pi \sum_{j=1}^k d_{ij}^2}

Where :math:`d_{ij}` is the distance between object :math:`i` and :math:`j`.
When querying for neighbors, we can either look at the object catalog when testing the pipeline or the truth catalog when testing for science.
There will likely be some underlying science that can be extracted by using the truth catalog density but we limit our focus to the detected catalog to test the pipeline.

The distribution of density and the relationship with unrecognized blends are shown below

.. image:: ./_static/obj_density.png

Figure 6. Log scale histogram of object density using 5 neighbors.


.. image:: ./_static/unrec_blend_density.png

Figure 7. Fraction of unrecognized blend as a function of local detected density (left) and local true density (right). As expected, the fraction of unrecognized blends monotonically increases with true density however the observed density flat-lines.

.. note:: 
    Removed the heatmaps section since I'm not sure what the actual take away is...

.. 
        Heatmaps
        ---------

        We also make some heatmaps to see how multiple variables interact.

        .. image:: ./_static/heatmap_e1_e2.png

        Figure 7. Fraction of unrecognized blend 


Conclusion
==========
    * We have created a set of tools that enable us to match between catalogs to label unrecognized blends and investigate how the rate of unrecognized blends vary with object properties.
    * This technote has the ideal case using simulated data along with true input catalogs which gives a good goalpost for comissioning data. 
    * During commissioning and observation we intend to re-do this analysis using space based data which will enable future studies on unrecognized blends and how to mitigate them.


