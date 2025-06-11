#############################################
Unrecognized Blends in ComCam ECDFS
#############################################

.. abstract::
   Unrecognized blends are a class of blended objects where two (or more) objects are so close on the sky that they are mistakenly identified as a single object.
   These objects can cause a variety of issues for science and simple validation. 
   We can identify such objects by using higher resolution imaging from space-based telescopes that are not affected by ground-based seeing and then label detected objects as isolated, recognized blends, or unrecognized blends.
   We find that for :math:`i > 23`, 15\% to 20\% of objects are unrecognized blends. 

.. We then investigate various factors that influence the fraction of unrecognized blends. 

Data
===============
..   * Introduce Operations Rehearsal 3 with 3 nights of simulation
   * We are using the "Intermittent Cumulative DRP" catalog

        * Also reference a nightly catalog/collection

   * We need to use the truth catalogs for matching
   * Truth catalogs use galaxies, stars, solar system objects and we are only looking at galaxies and stars
   * We apply the :code:`detect_isPrimary` flag which

        * Works with deblended children, removes sky objects, and is only inner regions
        * Might be a problem to apply cuts before matching but good for understandability
        * Double check!

   * In the observed catalog we only have :code:`extendedness` which is 1 for extended objects. We assume all extended objects are galaxies and use the two interchangibly 



.. The Commissioning Camera (ComCam) was a stand-in camera used by the Vera Rubin Observatory to test the adaptive optics system.

The Extended Chandra Deep Field-South (ECDFS), or also known as GOODS-South, is an extension to the original Chandra Deep Field-South which was originally done in X-Ray but has since been observed across many bands.
The Commissioning Camera (ComCam) was able to observe this patch of sky with over 1000 visits in total, 250 being in the :math:`i`-band alone, similar to 10-year depth.
This data was then processed several times through the Rubin pipelines enabling rapid improvement to the entire system.
Unrecognized blends allows us to understand some of the inherent failure modes of object detection. 
We use the :code:`/repo/dp1` repo and :code:`LSSTComCam/runs/DRP/DP1/v29_0_0/DM-50260` collection for ComCam data along with HST CANDELS data. 

The DP1 catalog includes a :code:`deblending` algorithm which means with accurate detection it is able to parse isolated and ``recognized blends.``
Deblending produces children objects from a parent object, both of which are in the catalog and needs to be pruned in order to remove duplicates. 
We apply the general :code:`detect_isPrimary` flag which removes the parent objects (if child object exist) from the catalog along with removing any sky objects and that the object is from the inner part of both a tract and a patch. 
We will use the terms "extended object" (defined by :code:`refExtendedness == 1`) and "observed galaxy" interchangibly.
We place two cuts on the HST catalog, that the F814 magnitude be greater than some tunable value which we call the space magnitude (:math:`m_s`) and that :code:`FLAG == 0` which removes only 318 objects.
Details on the HST :code:`FLAG` parameter can be found `here <https://archive.stsci.edu/hlsps/candels/goods-s/catalogs/v1/hlsp_candels_hst_wfc3_goodss-tot-multiband_f160w_v1_readme.pdf>`_. 

The overlap between the two catalogs can be seen in :numref:`overlap` and the magnitude distribution (:math:`i` for ComCam and F814 for HST) in :numref:`magdist`.

.. _overlap:
.. figure:: ./_static/hst_comcam_overlap.png

        Area of overlap between the two surveys.

.. _magdist:
.. figure:: ./_static/hst_comcam_magdist.png

        Log scale histogram of i-magnitude distribution for ComCam and F814-magnitude for HST. The dashed lines are the completeness limits of 25.4 and 26.5 respectively.


.. To label detected objects as isolated galaxies or unrecognized blends we require a higher resolution catalog which corresponds to space based data for actual operations and input truth for simulations.
   The truth catalogs for this run include galaxies, stars, and solar system objects of which we only consider the galaxies and stars.
   The input truth catalog for solar system objects does not include an *i*-magnitude and while they will cause blending, difference imaging is likely to offer much better mitigation of these sources than deblending.
   The simulated images are built on top of a set of DC2 patches with input catalogs in :code:`/sdf/data/rubin/shared/ops-rehearsals/ops-rehearsal-3/imSim_catalogs`. 

Matching
========
..   * We have to match between the space (truth) and ground catalogs to label unrecognized blends

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

Ground and space catalogs in hand, we can start to label objects in the ground catalog as isolated objects (pure), recognized blends, or unrecognized blends by matching between the two catalogs.
The general idea will be to generate a list of candidate unrecognized blends and then prune that list for the problematic unrecognized blends.
This includes removing pure and recognized blends, along with removing any unrecognized blends that are unlikely to be contaminated (a 23-mag blended with a 27 mag).


Seemingly the simplest way to match between the two catalogs would be to use pure spatial information, RA and DEC. 
Querying the ground and space catalogs within some :code:`search_radius` (which we choose to be the same for both catalogs) and then comparing the counts which can be done quickly using a k-d tree datastructure like the one implemented in `scipy <10.1038/s41592-019-0686-2.>`_ as :code:`scipy.spatial.kdtree`.
This can work well for pure objects but can lead to inconsistent results when varying the :code:`search_radius` parameter.
The purely spatial matching will also be inconsistent in labelling recognized blends properly. 
However, it is a common matcher so the results are included below but elect to pursue a more involed **ellipse matcher.**



The ellipse matcher uses the position (RA, DEC) and shape parameters (A, B, :math:`\Theta`) to model objects in both catalogs as ellipses and then require that there be overlap between the ground ellipse and space ellipse.
If there are multiple space ellipses overlapping with the same ground ellipse, that object is an unrecognized blend.
For DP1, the shape parameters come from :code:`shape_xx`, :code:`shape_xy`, and :code:`shape_yy` that can be converted to A, B, and :math:`\Theta` while the HST catalog includes these values as measured from Source Extractor.
Note that a pixel-to-arcsecond factor must be used to make sure that the two catalogs have meaningfully consistent units when we make comparisons.
The center of each object, measured by RA and DEC, with the above parameters, A, B, and :math:`\Theta`, can then be turned to the form of

.. math::

        Ax^2 + By^2 + Cxy + Dx + Ey + F = 0

which can be used in an analytic expression to determine if there is overlap with another ellipse.
On average, the product :math:`\sqrt{A \times B} \approx` the half-light radius so we scale both parameters by a :code:`candidate_boost_factor` which we will set to 2 unless otherwise specified.

For each ground object, we query for objects within 5'' of the original ground object in both the ground and space catalog and determine if there is any overlap in either catalog.
We query the ground catalog to account for recognized blends.
If there are more space objects (:math:`\hat{N_s}`) than ground objects (:math:`\hat{N_g}`), the object is an unrecognized blend.
If there are equal amounts then the object is either ``pure`` (:math:`\hat{N_s} = \hat{N_g} = 1`)) or ``recognized blend`` (:math:`\hat{N_s} = \hat{N_g} > 1`).
In the case of missing space data or spurious detections (:math:`\hat{N_g} > \hat{N_s}`), the object is ignored for subsequent analysis.

Once we have a set of candidate blends, we can apply magnitude cuts on these to isolate to the problematic unrecognized blends.
For each candidate, we require that the set of truth objects pass 2 cuts:

    #. The truth objects should not be too dim in band X :math:`m_X < m_s`
    #. The difference between a truth object and the brightest in the set in band X should be small :math:`\Delta_X < m_\Delta`

The first cut can be understood as requiring completeness in the space catalog with the second restricting our search to blends that are likely to have an impact on measured properties such as flux and shape. 
For HST, we will focus on the F814 band.
Once applying the cuts on the truth catalog we recount the number of objects in the sets and promote any surviving candidate unrecognized blends to candidate blends.
In summary we have the following process: 

- For each ground detection

        #.  Querying a radius :math:`r` around the ground detection RA and DEC in the ground catalog gives :math:`\hat{N_g}` ground objects.
        #.  Querying a radius :math:`r` around the ground detection RA and DEC in the space catalog gives :math:`\hat{N_s}` space objects.
        #.  Each of the :math:`\hat{N_g}` objects are converted into ellipses and check for overlap with the ground detection leaving :math:`N_g` objects.
        #.  Each of the :math:`\hat{N_s}` objects are converted into ellipses and check for overlap with the ground detection leaving :math:`N_s` objects.
        #.  If :math:`N_s > N_g` we have a **candidate unrecognized blend**.
        #.  If :math:`N_s \leq N_g` we have a *recognized blend* or a *spurious detection*.

- Promoting a *candidate unrecognized blend* requires the space objects to pass 2 cuts:

        * :math:`m_{F814} < m_s`.
        * :math:`\Delta_{F814} < m_\Delta`.


In total, we use two matching algorithms, :code:`ellipse` and :code:`spatial` to generate a list of candidate blends and then refine using the same magnitude requirements.. 

.. Another option is to process the candidates through the :code:`blend-entropy` scheme which can weight each blend based on distance, overlap, and magnitude of objects.
   This is a robust scheme that allows for handling of both unrecognized blends and problematic recognized blends but quite slow due to the computational overhead. 


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

Using the matching schemes detailed above we can label isolated objects (pure), recognized blends and unrecognized blends in the ground catalog.
Any other object is left out of the catalog.
Unless otherwise specified, we set :code:`candidate_boost_factor = 2`, :math:`m_s = 26.5`, and :math:`m_\Delta = 2`.
Due to setting :math:`m_\Delta = 2`, even though the ComCam :math:`i`-mag distribution peaks at 25, we are only able to confidently label blends up to :math:`m_g = 24.5`.
When relevant, the simpler KDTree method will also be presented showing results using :code:`search_radius = candidate_boost_factor`.


Magnitude Dependence
--------------------------
We expect that blending will increase at the higher magnitudes as dimmer objects are harder to uniformly detect and there are simply more galaxies.
The fraction of unrecognized blends as a function of the observed *i*-mag is shown with a comparison between the two methods included.
Restricting to only extended objects -- observed galaxies -- produces almost no change in the distribution of unrecognized blends.

.. figure:: _static/unrec_blend_imag.png

   Fraction of unrecognized blends as a function of observed *i-mag*. Ellipse matching results are shown in blue and pure spatial results using blend entropy in orange. The two show a similar bump at the faint end while there is not strict agreement.


In comparison to the Roman-Rubin simulations by `Troxel et al. <https://arxiv.org/pdf/2209.06829>`_, we see find similar levels of blending when using purely spatial matching but not with the ellipse matching. 

..  If we restrict to only detected galaxies we see a slight increase
        .. figure:: ./_static/unrec_blend_extended.png

            Fraction of unrecognized blend as a function of observed i-mag including a restriction on only observed galaxies. There is a slight increase corresponding to extended objects being easier to overlap with.

Shape Parameters
-----------------
Accurate shape measurements is necessary for weak lensing studies and it is expected that unrecognized blends will impact shear.
The inverse question, if certain shapes will impact unrecognized blends is not as well studied.
It is possible that there would be a bias due to the orientation of the pixel grid which we investigate below.

We look at the second moment, :math:`Q_{ij}`, of extended objects.
We combine the second moments via 

.. math::
   e_1 = \frac{Q_{xx} - Q_{yy}}{Q_{xx} + Q_{yy}} \;\;\; e_2 = \frac{2Q_{xy}}{Q_{xx} + Q_{yy}}

..
        We create :math:`Q_{rr} = \sqrt{Q_{xx}^2 + Q_{yy}^2}`.

        .. figure:: ./_static/unrec_blend_shapeij.png
           
                Fraction of unrecognized blend as a function of measured second moments on observed galaxies. The range is limited to the 95% range for each measurement.

.. figure:: ./_static/unrec_blend_pol.png

        Fraction of unrecognized blend as a function of ellipse polarization on observed galaxies.

Given that there is little to no difference among the shape parameters, this gives good confidence that the pixel grid is not impacting shape measurements and unrecognized blends in strange ways.
The wing structure is not necessarily cause for concern but it is interesting that objects with larger polarization is correlated with to be unrecognized blends.

Local Density
--------------
Finally, we know clusters and other dense fields (like the deep fields) are expected to be extremely blended which motivates looking into how local density affects unrecognized blend fraction.

To estimate the local density, :math:`\sum(r_i)`, we use Equation 7 from `Darvish et al <https://arxiv.org/pdf/1503.07879.pdf>`_.

.. math::
   \sum(r_i) = \frac{\sum_{j=1}^k j}{\pi \sum_{j=1}^k d_{ij}^2}

Where :math:`d_{ij}` is the distance between object :math:`i` and :math:`j` and :math:`k` is the number of neighbors which we set to 5.
We look at both the ground and space based catalog and calculate two independent densities.

The relationship of unrecognized blends and local density are shown below


.. figure:: ./_static/unrec_blend_density.png

        Fraction of unrecognized blend as a function of ComCam object density (blue) and HST object density (orange). 

As expected, the fraction of unrecognized blends monotonically increases with HST density; however, rather unexpectedly, we see that the blending rate decreases with the ground based ComCam density.


.. note:: 
    What the heck is happening with the comcam density?! Something to do with the deblender doing well?

.. 

Conclusion
==========

We have outlined a matching scheme that allows for robustly classifying objects as isolated, recognized blends and unrecognized blends.
Using the ellipse matching method, we investigate the occurance of unrecognized blends in the ComCam ECDFS data and how it varies with severalproperties like *i*-mag and local density.
Comparing to simulations like the Roman-Rubin simulation, we find similar rates of unrecognized blends versus *i*-magnitude when using purely spatial matching but not ellipse matching.
In total, unrecognized blends in ComCam is at the expected levels and not suffering from pipeline issues.
