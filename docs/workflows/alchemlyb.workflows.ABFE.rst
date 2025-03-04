The ABFE workflow
==================
The *Absolute binding free energy* (ABFE) workflow provides a complete workflow
that uses the energy files generated by MD engine as input and generates the
binding free energy as well as the analysis plots.

Fully Automatic analysis
------------------------
*Absolute binding free energy* (ABFE) calculations can be analyzed with
two lines of code in a fully automated manner (similar to 
`Alchemical Analysis <https://github.com/MobleyLab/alchemical-analysis>`_).
In this case, any parameters are set when invoking :class:`~alchemlyb.workflows.abfe.ABFE`
and reasonable defaults are chosen for any parameters not set explicitly. The two steps
are to

1. initialize an instance of the :class:`~alchemlyb.workflows.abfe.ABFE` class
2. invoke the :meth:`~alchemlyb.workflows.ABFE.run` method to execute
   complete workflow.
   
For a GROMACS ABFE simulation, executing the workflow would look similar
to the following code (The log is configured by logger). ::

    >>> from alchemtest.gmx import load_ABFE
    >>> from alchemlyb.workflows import ABFE
    >>> # Enable the logger
    >>> import logging
    >>> logging.basicConfig(filename='ABFE.log', level=logging.INFO)
    >>> # Obtain the path of the data
    >>> import os
    >>> dir = os.path.dirname(load_ABFE()['data']['complex'][0])
    >>> print(dir)
    'alchemtest/gmx/ABFE/complex'
    >>> workflow = ABFE(units='kcal/mol', software='GROMACS', dir=dir,
    >>>                 prefix='dhdl', suffix='xvg', T=298, outdirectory='./')
    >>> workflow.run(skiptime=10, uncorr='dhdl', threshold=50,
    >>>              methods=('MBAR', 'BAR', 'TI'), overlap='O_MBAR.pdf',
    >>>              breakdown=True, forwrev=10)


The workflow uses the :class:`~alchemlyb.parsing` to parse the data from the
energy files, remove the initial unequilibrated frames and decorrelate the data
with :class:`~alchemlyb.preprocessing.subsampling`. The decorrelated dataset
:ref:`dHdl <dHdl>` and :ref:`u_nk <u_nk>` are then passed to
:class:`~alchemlyb.estimators` for free energy estimation. The workflow will
also perform a set of analysis that allows the user to examine the quality of
the estimation.

File Input
^^^^^^^^^^

This command expects the energy files to be structured in two common ways. It
could either be ::

    simulation
    ├── lambda_0
    │   ├── prod.xvg
    │   └── ...
    ├── lambda_1
    │   ├── prod.xvg
    │   └── ...
    └── ...

Where :code:`dir='simulation/lambda_*', prefix='prod', suffix='xvg'`. Or ::

    dhdl_files
    ├── dhdl_0.xvg
    ├── dhdl_1.xvg
    └── ...

Where :code:`dir='dhdl_files', prefix='dhdl_', suffix='xvg'`.

Output
^^^^^^

The workflow returns the free energy estimate using all of
:class:`~alchemlyb.estimators.TI`, :class:`~alchemlyb.estimators.BAR`,
:class:`~alchemlyb.estimators.MBAR`. For ABFE calculations, the alchemical
transformation is usually done is three stages, the *bonded*, *coul* and *vdw*
which corresponds to the free energy contribution from applying the
restraint to restrain the ligand to the protein, decouple/annihilate the
coulombic interaction between the ligand and the protein and
decouple/annihilate the protein-ligand lennard jones interactions. The result
will be stored in :attr:`~alchemlyb.workflows.ABFE.summary` as
:class:`pandas.Dataframe`. ::


                          MBAR  MBAR_Error        BAR  BAR_Error         TI  TI_Error
    States 0 -- 1     0.065967    0.001293   0.066544   0.001661   0.066663  0.001675
           1 -- 2     0.089774    0.001398   0.089303   0.002101   0.089566  0.002144
           2 -- 3     0.132036    0.001638   0.132687   0.002990   0.133292  0.003055
    ...
           26 -- 27   1.243745    0.011239   1.245873   0.015711   1.248959  0.015762
           27 -- 28   1.128429    0.012859   1.124554   0.016999   1.121892  0.016962
           28 -- 29   1.010313    0.016442   1.005444   0.017692   1.019747  0.017257
    Stages coul      10.215658    0.033903  10.017838   0.041839  10.017854  0.048744
           vdw       22.547489    0.098699  22.501150   0.060092  22.542936  0.106723
           bonded     2.374144    0.014995   2.341631   0.005507   2.363828  0.021078
           TOTAL     35.137291    0.103580  34.860619   0.087022  34.924618  0.119206

Output Files
^^^^^^^^^^^^

For quality assessment, a couple of plots were generated and written to
the folder specified by `outdirectory`.

The :ref:`overlay matrix for the MBAR estimator <plot_overlap_matrix>` will be
plotted and saved to :file:`O_MBAR.pdf`, which examines the overlap between
different lambda windows.

The :ref:`dHdl for TI <plot_TI_dhdl>` will be plotted to
:file:`dhdl_TI.pdf`, allows one to examine if the lambda scheduling has
covered the change of the gradient in the lambda space.

The :ref:`dF states <plot_dF_states>` will be plotted to :file:`dF_state.pdf` in
portrait model and :file:`dF_state_long.pdf` in landscape model, which
allows the user to example the contributions from each lambda window.

The forward and backward convergence will be plotted to :file:`dF_t.pdf` using
:class:`~alchemlyb.estimators.MBAR` and saved in
:attr:`~alchemlyb.workflows.ABFE.convergence`, which allows the user to
examine if the simulation time is enough to achieve a converged result.

Semi-automatic analysis
-----------------------
The same analysis could also performed in steps allowing access and modification
to the data generated at each stage of the analysis. ::

    >>> from alchemtest.gmx import load_ABFE
    >>> from alchemlyb.workflows import ABFE
    >>> # Obtain the path of the data
    >>> import os
    >>> dir = os.path.dirname(load_ABFE()['data']['complex'][0])
    >>> print(dir)
    'alchemtest/gmx/ABFE/complex'
    >>> # Load the data
    >>> workflow = ABFE(software='GROMACS', dir=dir,
    >>>                 prefix='dhdl', suffix='xvg', T=298, outdirectory='./')
    >>> # Set the unit.
    >>> workflow.update_units('kcal/mol')
    >>> # Read the data
    >>> workflow.read()
    >>> # Decorrelate the data.
    >>> workflow.preprocess(skiptime=10, uncorr='dhdl', threshold=50)
    >>> # Run the estimator
    >>> workflow.estimate(methods=('mbar', 'bar', 'ti'))
    >>> # Retrieve the result
    >>> summary = workflow.generate_result()
    >>> # Plot the overlap matrix
    >>> workflow.plot_overlap_matrix(overlap='O_MBAR.pdf')
    >>> # Plot the dHdl for TI
    >>> workflow.plot_ti_dhdl(dhdl_TI='dhdl_TI.pdf')
    >>> # Plot the dF states
    >>> workflow.plot_dF_state(dF_state='dF_state.pdf')
    >>> # Convergence analysis
    >>> workflow.check_convergence(10, dF_t='dF_t.pdf')

API Reference
-------------
.. autoclass:: alchemlyb.workflows.ABFE
    :members:
    :inherited-members: