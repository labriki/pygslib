
Tutorial: Resource estimation with PyGSLIB
==========================================

This tutorial will guide you on doing resource estimation with PyGSLIB. The informing data is from the `BABBITT zone of the
KEWEENAWAN DULUTH
COMPLEX <http://www.nrri.umn.edu/egg/REPORTS/TR200321/TR200321.html>`__.

The sequence of data preparation and estimation used in this example is
as follows:

-  import drillhole tables
-  create a drillhole object
-  composite drillhole intervals
-  desurvey drillholes
-  code drillholes with domains
-  declustering and basic statistics
-  create a block model
-  populate a wireframe domain with blocks
-  estimate with ordinary kriging
-  basic model validation

To run this demo in your computer download and unpack `the demo
files <_files/PyGSLIB_Tutorial1.zip>`__. If you have issues installing
PyGSLIB `follow this
tutorial <https://www.youtube.com/watch?v=cbWXi7BfZVg>`__

Preliminaries
-------------

To follow this tutorial you may be familiar with:

-  Python 2.7
-  Python Data Analysis Library (Pandas). PyGSLIB was designed to work
   with Pandas.
-  Numpy
-  Matplotlib
-  Paraview, an open-source, multi-platform data analysis and
   visualization application
-  GSLIB, geostatistics and mineral resource estimation

Loading python packages
-----------------------

.. code:: python

    >>> # import third party python libraries
    >>> import pandas as pd
    >>> import matplotlib.pylab as plt
    >>> import numpy as np
    >>> # make plots inline in Ipython Notebooks/QT terminal
    >>> # %matplotlib inline
    

.. code:: python

    >>> # import pygslib
    >>> import pygslib
    


Need some help? Just type

::

    print pygslib.__doc__

or go to `the PyGSLIB online
manual <https://opengeostat.github.io/pygslib/>`__

Loading the drillhole tables
----------------------------

PyGSLIB only understands drillholes tables loaded into Pandas
DataFrames. Two tables are compulsory: 

- the collar, with the compulsory fields *BHID*, *XCOLLAR*, *YCOLLAR* and *ZCOLLAR* and the optional field *LENGTH* 
- and survey, with the compulsory fields *BHID*, *AT*, *AZ*, *DIP*

In addition, you may have any number of optional interval tables with
the compulsory fields *BHID*, *FROM* and *TO*

.. code:: python

    >>> # importing data from file
    >>> collar = pd.read_csv('collar_BABBITT.csv')
    >>> survey = pd.read_csv('survey_BABBITT.csv')
    >>> assay = pd.read_csv('assay_BABBITT.csv')

    
``collar``, ``survey`` and ``assay`` are loaded on Pandas DataFrames
objects

.. code:: python

    >>> print type(collar)
    
    <class 'pandas.core.frame.DataFrame'>
    
    

You can see the columns types and plot the tables as follow:

.. code:: python

    >>> print collar.dtypes
    
    BHID        object
    XCOLLAR    float64
    YCOLLAR    float64
    ZCOLLAR    float64
    dtype: object
    


.. code:: python

    >>> print collar.head(3)
    
         BHID     XCOLLAR    YCOLLAR  ZCOLLAR
    0   34873  2296021.09  414095.85   1590.0
    1  B1-001  2294148.20  420495.90   1620.9
    2  B1-002  2296769.50  422333.50   1553.0
    
    
    



.. code:: python

    >>> print survey.head(3)
    
         BHID   AT   AZ   DIP
    0   34873  0.0    0  90.0
    1  B1-001  0.0  327  60.0
    2  B1-002  0.0  327  60.0
    

.. code:: python

    >>> print assay.head(3)
    
        BHID    FROM      TO    CU    NI   S  FE
    0  34873     0.0  2515.0   NaN   NaN NaN NaN
    1  34873  2515.0  2517.4  0.03  0.08 NaN NaN
    2  34873  2517.4  2518.9  0.04  0.10 NaN NaN
    
    



Pandas provides a large set of functions to modify your data. Lets
remove some columns and make non-assayed intervals equal to zero.

.. code:: python

    >>> # droping some columns
    >>> assay.drop(['NI','S','FE'], axis=1, inplace=True)
    >>> # making non-sampled intervals equal to zero
    >>> assay.loc[~np.isfinite(assay['CU']), 'CU']=0


    
Creating drillhole object
-------------------------

To get access to the drillhole functions implemented in PyGSLIB, such as
desurvey and compositing, you need to create a drillhole object (an
instance of the class ``Drillhole``, defined on the submodule
``gslib.drillhole``)

.. code:: python

    >>> #creating a drillhole object
    >>> mydholedb=pygslib.drillhole.Drillhole(collar=collar, survey=survey)
    >>> # now you can add as many interval tables as you want, for example, assays, lithology and RQD.
    >>> mydholedb.addtable(assay, 'assay', overwrite = False)
    
    Warning (from warnings module):
      File "C:/OG_Python/pygslib_database/DULUTH/Babbitt_example/test.py", line 68
        mydholedb=pygslib.drillhole.Drillhole(collar=collar, survey=survey)
    UserWarning: ! Collar table without LENGTH field


The output above is a warning message. This one is a complain because
the field ``LENGTH`` was not included in the collar table. You will see
similar warnings any time PyGSLIB detects a potential issue in your
data.

.. code:: python

    >>> # validating a drillhole object
    >>> mydholedb.validate()
    
    Warning (from warnings module):
      File "C:/OG_Python/pygslib_database/DULUTH/Babbitt_example/test.py", line 74
        mydholedb.validate()
    UserWarning: ! survey with one value at BHID: 34873. This will produce error at desurvey

    Warning (from warnings module):
      File "C:/OG_Python/pygslib_database/DULUTH/Babbitt_example/test.py", line 74
        mydholedb.validate()
    UserWarning: ! survey with one value at BHID: B1-001. This will produce error at desurvey
    

The warning above is serious. There are drillholes with only one survey record and to desurvey we need at least two records, the first one may be at the collar of the drillhole. 

.. code:: python

    >>> # fixing the issue of single interval at survey table
    >>> mydholedb.fix_survey_one_interval_err(90000.)

Note: To validate interval tables you may use the function
``validate_table``.

.. code:: python

    >>> # validating interval tables
    >>> mydholedb.validate_table('assay')

Compositing
-----------

Before doing any statistical or geostatistical analysis you may verify
that all samples have approximately the same length. If samples have
different lengths you may "resample" the drillhole intervals using a
compositing algorithm.

.. code:: python

    >>> # Calculating length of sample intervals
    >>> mydholedb.table['assay']['Length']= mydholedb.table['assay']['TO']- mydholedb.table['assay']['FROM']
    >>> # printing length mode
    >>> print 'The Length Mode is:', mydholedb.table['assay']['Length'].mode()[0]
    
    The Length Mode is: 10.0
    
    >>> # plotting the interval lengths
    >>> mydholedb.table['assay']['Length'].hist(bins=np.arange(15)+0.5)
    >>> plt.show()
   
.. image:: Tutorial_files/Tutorial_25_1.png


Most samples (the mode) are 10 ft. length. This value or any of its
multiples are good options of composite length, they minimize the
oversplitting of sample intervals.

.. code:: python

    >>> # compositing 
    >>> mydholedb.downh_composite('assay', variable_name= "CU", new_table_name= "CMP", 
    ...                            cint = 10, minlen=-1, overwrite = True)


.. code:: python

    >>> # first 5 rows of a table
    >>> print mydholedb.table["CMP"].tail(5)
    
                BHID   CU    FROM      TO  _acum  _len
    54184  RMC-66313  0.0   970.0   980.0    0.0  10.0
    54185  RMC-66313  0.0   980.0   990.0    0.0  10.0
    54186  RMC-66313  0.0   990.0  1000.0    0.0  10.0
    54187  RMC-66313  0.0  1000.0  1010.0    0.0  10.0
    54188  RMC-66313  0.0  1010.0  1020.0    0.0   7.0



Note that some especial fields were created, those fields have prefix
``_``. ``_acum`` is the grade accumulated in the composite interval (sum
of grades from sample intervals contributing to the composite interval)
and ``_len`` is the actual length of the composite.

In the table CMP the interval at row 54188 has *FROM : 1010.0* and *TO:
1020.0* but the sample length is only *7.0 ft*. In this way the *FROM*
and *TO* intervals of any drillhole or table are always at the same
position and you can safely use the fields *[BHID, FROM]* to merge
tables.

Desurveying
-----------

To plot drillholes in 3D or to estimate grade values you need to
calculate the coordinates of the composites. This process is known as
*desurvey*. There are many techniques to desurvey, PyGSLIB uses minimum
curvature.

Desurvey will add the fields ``azm, dipm`` and ``xm, ym, zm``, these are
directions and the coordinates at the mid point of composite intervals.
You have the option to add endpoint coordinates ``xb, yb, zb`` and
``xe, ye, ze``, these are required to export drillholes to Paraview (in
vtk format).

.. code:: python

    >>> # desurveying an interval table
    >>> mydholedb.desurvey('CMP',warns=False, endpoints=True)
    >>> # first 3 rows of a table
    >>> print mydholedb.table["CMP"].head(3)
    
        BHID   CU  FROM    TO  _acum  _len  azm  dipm         xm            ym  
    0  34873  0.0   0.0  10.0    0.0  10.0  0.0  90.0  2296021.0  414095.84375   
    1  34873  0.0  10.0  20.0    0.0  10.0  0.0  90.0  2296021.0  414095.84375   
    2  34873  0.0  20.0  30.0    0.0  10.0  0.0  90.0  2296021.0  414095.84375   
    
           zm         xb            yb      zb         xe            ye      ze  
    0  1585.0  2296021.0  414095.84375  1590.0  2296021.0  414095.84375  1580.0  
    1  1575.0  2296021.0  414095.84375  1580.0  2296021.0  414095.84375  1570.0  
    2  1565.0  2296021.0  414095.84375  1570.0  2296021.0  414095.84375  1560.0  
    
    

Creating a BHID of type integer
--------------------------------

The compiled FORTRAN code of GSLIB is not good with data of type *str*,
sometimes you need to transform the *BHID* to type *int*, for example,
if you use a maximum number of samples per drillholes on kriging. The
function ``txt2intID`` will do this work for you.

.. code:: python

    >>> # creating BHID of type integer
    >>> mydholedb.txt2intID('CMP')
    >>> # first 3 rows of a subtable
    >>> print mydholedb.table["CMP"][['BHID', 'BHIDint', 'FROM', 'TO']].tail(3)
    
                BHID  BHIDint    FROM      TO
    54186  RMC-66313      399   990.0  1000.0
    54187  RMC-66313      399  1000.0  1010.0
    54188  RMC-66313      399  1010.0  1020.0




Rendering drillhole intervals in Paraview and exporting drillhole data
----------------------------------------------------------------------

PyGSLIB can export drillhole intervals to VTK. Drag and drop the VTK
file on Paraview to see the drillholes in 3D. For a better image quality
add a *tube* filter and update the color scale.

.. code:: python

    >>> # exporting results to VTK
    >>> mydholedb.export_core_vtk_line('CMP', 'cmp.vtk', nanval=0, title = '')


This is how it looks in Paraview

.. figure:: Tutorial_files/figure1.JPG
   :alt: Drillhole 3D view

Interval tables are stored as a python dictionary of *{Table Name :
Pandas Dataframes}*. To export data to \*.csv format use the Pandas
function ``Dataframe.to_csv``. You can also export to any other format
supported by Pandas, `this is the list of formats
supported <http://pandas.pydata.org/pandas-docs/stable/io.html>`__.

.. code:: python

    >>> # inspecting interval tables in drillhole object
    >>> print "Table names ", mydholedb.table_mames
    ... print "Tables names", mydholedb.table.keys()
    ... print "Table is    ", type(mydholedb.table)
    
    Table names  ['assay', 'CMP']
    Tables names ['assay', 'CMP']
    Table is     <type 'dict'>

    

.. code:: python

    >>> # exporting to csv
    >>> mydholedb.table["CMP"].to_csv('cmp.csv', index=False)


Tagging samples with domain code
--------------------------------

Use the function ``pygslib.vtktools.pointinsolid`` to label
composites in a domain defined by a closed wireframe. You can
also use this function to label samples in open surfaces (ej. between two
surfaces), below a surface and above a surface.

.. code:: python

    >>> # importing the wireframe
    >>> domain=pygslib.vtktools.loadSTL('domain.stl')


Only Stereo Lithography (\*.STL) and XML VTK Polydata (VTP) file formats
are implemented. If your data is in a different format, ej. DXF, you can
use a file format converter, my favorite is
`meshconv <http://www.patrickmin.com/meshconv>`__

.. code:: python

    >>> # creating array to tag samples in domain1
    >>> inside1=pygslib.vtktools.pointinsolid(domain, 
    ...                       x=mydholedb.table['CMP']['xm'].values, 
    ...                       y=mydholedb.table['CMP']['ym'].values, 
    ...                       z=mydholedb.table['CMP']['zm'].values)
    >>> 
    >>> # creating a new domain field 
    >>> mydholedb.table['CMP']['Domain']=inside1.astype(int)
    >>> # first 3 rows of a subtable
    >>> print mydholedb.table['CMP'][['BHID', 'FROM', 'TO', 'Domain']].head(3)
    
        BHID  FROM    TO  Domain
    0  34873   0.0  10.0       0
    1  34873  10.0  20.0       0
    2  34873  20.0  30.0       0



.. code:: python

    >>> # exporting results to VTK
    >>> mydholedb.export_core_vtk_line('CMP', 'cmp.vtk', nanval=0, title = 'Generated with PyGSLIB')
    >>> # exporting to csv
    >>> mydholedb.table["CMP"].to_csv('cmp.csv', index=False)


A section of the wireframe and the drillholes may look as follows

.. figure:: Tutorial_files/figure4.JPG
   :alt: Drillhole tagging

Block modeling
--------------

Cu grades will be estimated on blocks inside the mineralized domain. To
create those blocks you may:

-  create a block model object ``pygslib.blockmodel.Blockmodel``
-  fill the mineralized domain with blocks

In PyGSLIB we use percent blocks, similar to GEMS. In the future we
will implement subcell style, similar to Surpac, using Adaptive Mesh
Refinement (AMR).

Blocks are stored in the class member ``bmtable``, this is a Pandas
DataFrame with especial field index ``IJK`` or ``[IX,IY,IZ]`` and
coordinates ``[XC, YC, ZC]``. We use GSLIB order, in other words,
``IJK`` is the equivalent of the row number in a GSLIB grid.

Block model tables can be full or partial (with some missing blocks).
Only one table will be available in a block model object.

The block model definition is stored in the members
``nx, ny, nz, xorg, yorg, zorg, dx, dy, dz``. The origin
``xorg, yorg, zorg`` refers to the lower left corner of the lower left
block (not the centroid), like in Datamine Studio.

.. code:: python

    >>> # The model definition
    >>> xorg = 2288230
    >>> yorg = 415200
    >>> zorg = -1000
    >>> dx = 100
    >>> dy = 100
    >>> dz = 30
    >>> nx = 160
    >>> ny = 100
    >>> nz = 90


.. code:: python

    >>> # Creating an empty block model
    >>> mymodel=pygslib.blockmodel.Blockmodel(nx,ny,nz,xorg,yorg,zorg,dx,dy,dz)


.. code:: python

    >>> # filling wireframe with blocks
    >>> mymodel.fillwireframe(domain)
    >>> # the fillwireframe function generates a field named  __in, 
    >>> # this is the proportion inside the wireframe. Here we rename __in to D1
    >>> mymodel.bmtable.rename(columns={'__in': 'D1'},inplace=True)
 

.. code:: python

    >>> # creating a partial model by filtering out blocks with zero proportion inside the solid
    >>> mymodel.set_blocks(mymodel.bmtable[mymodel.bmtable['D1']> 0])
    >>> # export partial model to a VTK unstructured grid (*.vtu)
    >>> mymodel.blocks2vtkUnstructuredGrid(path='model.vtu')


Note that ``fillwireframe`` created or overwrited ``mymodel.bmtable``.
The blocks outside the wireframe where filtered out and the final output
is a partial model with block inside or touching the wireframe domain.

Note that ``fillwireframe`` works with closed surfaces only.

A section view of the blocks colored by percentage inside the solid and
the wireframe (white lines) may look as follows:

.. figure:: Tutorial_files/figure3.JPG
   :alt: block model percentage

Some basic stats
----------------

You may spend some time doing exploratory data analysis, looking at
statistical plots, 3D views and 2D sections of your data. A good
comersial software for this is `Supervisor
<http://opengeostat.com/software-solutions/>`__, open source options
are Pandas, `Statsmodels <http://statsmodels.sourceforge.net/>`__,
`Seaborn <https://stanford.edu/~mwaskom/software/seaborn/>`__ and
`glueviz <http://glueviz.org/en/stable/>`__.

PyGSLIB includes some minimum functionality for statistical plots and
calculations, with support for declustering wight. Here we demonstrate
how you can do a declustering analysis of the samples in the mineralized
domain and how to evaluate the declustered mean. The declustered mean
will be compared later with the mean of CU estimates.

Note: In this section we are not including all the statistical analysis
usually required for resource estimation.

.. code:: python

    >>> #declustering parameters 
    >>> parameters_declus = { 
    ...        'x'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'xm'], 
    ...        'y'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'ym'],  
    ...        'z'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'zm'], 
    ...        'vr'     :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'CU'],   
    ...        'anisy'  :  1.,       
    ...        'anisz'  :  0.05,              
    ...        'minmax' :  0,                 
    ...        'ncell'  :  100,                  
    ...        'cmin'   :  100., 
    ...        'cmax'   :  5000.,                 
    ...        'noff'   :  8,                    
    ...        'maxcel' :  -1}
    >>> # declustering 
    >>> wtopt,vrop,wtmin,wtmax,error, \
    ...  xinc,yinc,zinc,rxcs,rycs,rzcs,rvrcr = pygslib.gslib.declus(parameters_declus)
    >>> #Plotting declustering optimization results
    >>> plt.plot (rxcs, rvrcr, '-o')
    >>> plt.xlabel('X cell size')
    >>> plt.ylabel('declustered mean')
    >>> plt.show()
    >>> plt.plot (rycs, rvrcr, '-o')
    >>> plt.xlabel('Y cell size')
    >>> plt.ylabel('declustered mean')
    >>> plt.show()
    >>> plt.plot (rzcs, rvrcr, '-o')
    >>> plt.xlabel('Z cell size')
    >>> plt.ylabel('declustered mean')
    >>> plt.show()




.. image:: Tutorial_files/Tutorial_53_0.png



.. image:: Tutorial_files/Tutorial_53_1.png



.. image:: Tutorial_files/Tutorial_53_2.png


.. code:: python

    >>> # parameters for declustering with the cell size selected
    >>> parameters_declus = { 
    ...        'x'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'xm'], 
    ...        'y'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'ym'],  
    ...        'z'      :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'zm'], 
    ...        'vr'     :  mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'CU'],  
    ...        'anisy'  :  1.,    # y == x
    ...        'anisz'  :  0.1,  # z = x/20     
    ...        'minmax' :  0,                 
    ...        'ncell'  :  1,                  
    ...        'cmin'   :  1000., 
    ...        'cmax'   :  1000.,                 
    ...        'noff'   :  8,                    
    ...        'maxcel' :  -1}  
    >>>  
    >>> 
    >>> # declustering 
    >>> wtopt,vrop,wtmin,wtmax,error, \
    ...   xinc,yinc,zinc,rxcs,rycs,rzcs,rvrcr = pygslib.gslib.declus(parameters_declus)
    >>> 
    >>> # Adding declustering weight to a drillhole interval table
    >>> mydholedb.table["CMP"]['declustwt'] = 1
    >>> mydholedb.table["CMP"].loc[mydholedb.table['CMP']['Domain']==1, 'declustwt'] = wtopt
    >>> # calculating declustered mean
    >>> decl_mean = rvrcr[0]


Estimating Cu grade in one block
--------------------------------

For estimation you may use the function ``pygslib.gslib.kt3d``, which is
the GSLIB's KT3D program modified and embedded into python. KT3D now
includes a maximum number of samples per drillhole in the search
ellipsoid and the estimation is only in the blocks provided as arrays.

The input parameters of ``pygslib.gslib.kt3d`` are defined in a large
and complicated dictionary. You can get this dictionary by typing

::

    print pygslib.gslib.kt3d.__doc__

Note that some parameters are optional. PyGSLIB will initialize those
parameters to zero or to array of zeros, for example if you exclude the
coordinate Z, PyGSLIB will create an array of zeros in its place.

To understand GSLIB's KT3D parameters you may read the `GSLIB user
manual <https://www.amazon.ca/GSLIB-Geostatistical-Software-Library-Users/dp/0195100158>`__
or `the kt3d gslib program parameter
documentation <http://www.statios.com/help/kt3d.html>`__.

Note that in PyGSLIB the parameters nx, ny and nz are only used by
superblock search algorithm, if these parameters are arbitrary the
output will be correct but the running time may be longer.

.. code:: python

    >>> # creating parameter dictionary for estimation in one block
    >>> kt3d_Parameters = {
    ...            # Input Data (Only using intervals in the mineralized domain)
    ...            # ----------
    ...            'x' : mydholedb.table["CMP"]['xm'][mydholedb.table["CMP"]['Domain']==1].values, 
    ...            'y' : mydholedb.table["CMP"]['ym'][mydholedb.table["CMP"]['Domain']==1].values,
    ...            'z' : mydholedb.table["CMP"]['zm'][mydholedb.table["CMP"]['Domain']==1].values,
    ...            'vr' : mydholedb.table["CMP"]['CU'][mydholedb.table["CMP"]['Domain']==1].values,
    ...            'bhid' : mydholedb.table["CMP"]['BHIDint'][mydholedb.table["CMP"]['Domain']==1].values, # an integer BHID
    ...            # Output (Target) 
    ...            # ----------
    ...            'nx' : nx,  
    ...            'ny' : ny,  
    ...            'nz' : nz, 
    ...            'xmn' : xorg,  
    ...            'ymn' : yorg,  
    ...            'zmn' : zorg,  
    ...            'xsiz' : dx,  
    ...            'ysiz' : dy,   
    ...            'zsiz' : dz, 
    ...            'nxdis' : 5,  
    ...            'nydis' : 5,  
    ...            'nzdis' : 3,  
    ...            'outx' : mymodel.bmtable['XC'][mymodel.bmtable['IJK']==1149229].values,  # filter to estimate only on block with IJK 1149229
    ...            'outy' : mymodel.bmtable['YC'][mymodel.bmtable['IJK']==1149229].values,
    ...            'outz' : mymodel.bmtable['ZC'][mymodel.bmtable['IJK']==1149229].values,
    ...            # Search parameters 
    ...            # ----------
    ...            'radius'     : 850,   
    ...            'radius1'    : 850,   
    ...            'radius2'    : 250,   
    ...            'sang1'      : -28,  
    ...            'sang2'      : 34,   
    ...            'sang3'      : 7,   
    ...            'ndmax'      : 12,    
    ...            'ndmin'      : 4,  
    ...            'noct'       : 0,
    ...            'nbhid'      : 3,   
    ...            # Kriging parameters and options 
    ...            # ----------
    ...            'ktype'      : 1,   # 1 Ordinary kriging 
    ...            'idbg'       : 1,   # 0 no debug 
    ...            # Variogram parameters 
    ...            # ----------
    ...            'c0'         : 0.35,    
    ...            'it'         : [2,2],    
    ...            'cc'         : [0.41,0.23], 
    ...            'aa'         : [96,1117],   
    ...            'aa1'        : [96,1117],  
    ...            'aa2'        : [96,300],   
    ...            'ang1'       : [-28,-28],   
    ...            'ang2'       : [ 34, 34],  
    ...            'ang3'       : [  7,  7]} 


The variogram model was modeled with `Supervisor
<http://opengeostat.com/software-solutions/>`__.

.. figure:: Tutorial_files/figure5.JPG
   :alt: Variograms

The variogram types are as explained in
http://www.gslib.com/gslib\_help/vmtype.html, for example,
``'it' : [2,2]`` means two exponential models, in other words
``[Exponential 1,Exponential 2]``

Only the block with index *IJK* equal to 1149229 was used this time and
``'idbg'`` was set to one in order to get a full output of the last (and
unique) block estimate, including the samples selected, kriging weight
and the search ellipsoid.

.. code:: python

    >>> # estimating in one block
    >>> estimate, debug, summary = pygslib.gslib.kt3d(kt3d_Parameters)
    



.. image:: Tutorial_files/Tutorial_58_0.png


.. code:: python

    >>> # saving debug to a csv file using Pandas
    >>> pd.DataFrame({'x':debug['dbgxdat'],'y':debug['dbgydat'],'z':debug['dbgzdat'],'wt':debug['dbgwt']}).to_csv('dbg_data.csv', index=False)
    >>> #pd.DataFrame({'x':[debug['dbgxtg']],'y':[debug['dbgytg']],'z':[debug['dbgztg']],'na':[debug['na']]}).to_csv('dbg_target.csv', index=False)
    >>> # save the search ellipse to a VTK file
    >>> pygslib.vtktools.SavePolydata(debug['ellipsoid'], 'search_ellipsoid')


The results may look like this in Paraview.

.. figure:: Tutorial_files/figure6.JPG
   :alt: Ellipsoid

Estimating in all blocks
------------------------

After testing the estimation parameters in few blocks you may be ready
to estimate in all the blocks within the mineralized domain. Just update
the parameter file to remove the debug option and reassign the target
coordinates as the actual blocks coordinate arrays.

.. code:: python

    >>> # update parameter file
    >>> kt3d_Parameters['idbg'] = 0 # set the debug of
    >>> kt3d_Parameters['outx'] = mymodel.bmtable['XC'].values  # use all the blocks 
    >>> kt3d_Parameters['outy'] = mymodel.bmtable['YC'].values
    >>> kt3d_Parameters['outz'] = mymodel.bmtable['ZC'].values


.. code:: python

    >>> # estimating in all blocks
    >>> estimate, debug, summary = pygslib.gslib.kt3d(kt3d_Parameters) 
    >>> # adding the estimate into the model
    >>> mymodel.bmtable['CU_OK'] = estimate['outest']


.. code:: python

    >>> # exporting block model to VTK (unstructured grid) 
    >>> mymodel.blocks2vtkUnstructuredGrid(path='model.vtu')
    >>> # exporting to csv using Pandas
    >>> mymodel.bmtable['Domain']= 1
    >>> mymodel.bmtable[mymodel.bmtable['CU_OK'].notnull()].to_csv('model.csv', index = False)


Validating the results
----------------------

There are few validations you may do:

-  visual validation
-  comparison of mean grade
-  swath plots
-  global change of support (GCOS)

Swath plots and GCOS are not implemented in PyGSLIB. For visual
validations you can use Paraview, for example:

.. figure:: Tutorial_files/figure7.JPG
   :alt: Visual validation

.. code:: python

    >>> print "Mean in model   :",  mymodel.bmtable['CU_OK'].mean()
    ... print "Mean in data    :", mydholedb.table["CMP"]['CU'][mydholedb.table["CMP"]['Domain']==1].mean()
    ... print "Declustered mean:", decl_mean
    
    Mean in model   : 0.2942070961
    Mean in data    : 0.366458176447
    Declustered mean: 0.327602770625
    


    

The swath plots and the global change of support were calculated with
`Supervisor <http://opengeostat.com/software-solutions/>`__. These
validation tests show that the model is not good enough.

To fix the estimate you can rerun this Ipython Notebook after changing
the parameters. Repeat the process until the estimate validates
properly.

.. figure:: Tutorial_files/figure8.JPG
   :alt: Validation in Supervisor

You may also try to rotate and the data before doing swath plots (use
the function ``pygslib.gslib.rotscale``). In addition you can try a
different estimator, for example MIK. The IK3D program is not
implemented in PyGSLIB but you can estimate indicators with
``pygslib.gslib.kt3d`` and post-process it with ``pygslib.gslib.postik``.

