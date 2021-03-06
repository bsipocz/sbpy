Activity Module (`sbpy.activity`)
=================================

Introduction
------------

`sbpy.activity` models cometary dust and gas activity.

Afρ and εfρ
-----------

`sbpy` has two classes to assist with observations and modeling of coma continuum: `Afrho` and `Efrho`.

The Afρ parameter of A'Hearn et al (1984) is based on observations of idealized cometary dust comae.  It is proportional to the observed flux density within a circular aperture.  The quantity is the product of dust albedo, dust filling factor, and the radius of the aperture at the distance of the comet.  It carries the units of ρ (length), and under certain assumptions is proportional to the dust production rate of the comet.  See A'Hearn et al. (1984) and Fink & Rubin (2012) for more details.  The εfρ parameter is the thermal emission counterpart to Afρ, replacing albedo with IR emissivity, defined by Kelley et al. (2013).

Afρ and εfρ are quantities
^^^^^^^^^^^^^^^^^^^^^^^^^^

`Afrho` and `Efrho` are subclasses of `astropy`'s `Quantity` and carry units of length.

  >>> import numpy as np
  >>> import astropy.units as u
  >>> from sbpy.activity import Afrho, Efrho
  >>> 
  >>> afrho = Afrho(100 * u.cm)
  >>> print(afrho)
  100.0 cm
  >>> efrho = Efrho(afrho * 3.5)
  >>> print(efrho)
  350.0 cm

They may be converted to any unit of length, and have convenience attributes that can be used for unit conversion.  Note these convenience attributes do not carry the units.

  >>> print('''
  ... Convert to meters = {}
  ...   Value as meters = {}
  ...       Value as au = {}
  ... '''.format(afrho.to('m'), afrho.m, afrho.au))
  Convert to meters = 1.0 m
    Value as meters = 1.0
        Value as au = 6.684587122268446e-12

Flux density
^^^^^^^^^^^^
	
The quantities may be initialized from flux densities.  Here, we reproduce one of the calculations from the original A'Hearn et al. (1984) work:

* $r_h=4.785$ au
* $\Delta=3.822$ au
* phase angle = 3.3°
* aperture radius = 9.8", 27200 km
* $\log{F\_\lambda}=-13.99$ erg/(cm2 s Å) at $\lambda=5240$ Å

The solar flux density at 1 au is also needed.  We use 1868 W/(m2 μm).

  >>> wave = 5240 * u.AA
  >>> flam = 10**-13.99 * u.Unit('erg/(s cm2 AA)')
  >>> aper = 27200 * u.km
  >>> eph = dict(rh=4.785 * u.au, delta=3.822 * u.au)
  >>> Slam = 1868 * u.W / u.m**2 / u.um
  >>> afrho = Afrho.from_fluxd(wave, flam, aper, eph=eph, S=Slam)
  >>> print(afrho)
  6029.90248952895 cm

Which is within a few percent of 6160 cm computed by A'Hearn et al.. The difference is likely due to the assumed solar flux density in the bandpass.

The `Afrho` class may be converted to a flux density, and the original value is recovered.

  >>> f = afrho.fluxd(wave, aper, eph=eph, S=Slam).to('erg/(s cm2 AA)')
  >>> print('''     F_λ = {}
  ... log(F_λ) = {}'''.format(f, np.log10(f.value)))
  F_λ = 1.0232929922807537e-14 erg / (Angstrom cm2 s)
  log(F_λ) = -13.99

The `fluxd` and `from_fluxd` methods work with units of flux density per wavelength or frequency.

  >>> fnu = flam.to('Jy', u.spectral_density(wave))
  >>> print(fnu)
  0.009372206976883997 Jy
  >>> Snu = 1.711e14 * u.Jy
  >>> print(Afrho.from_fluxd(wave, fnu, aper, eph=eph, S=Snu))
  6029.468388208903 cm

`Afrho` works seamlessly with `sbpy`'s calibration framework when the astropy affiliated package `synphot` is installed.  To convert to flux density using the default solar spectrum omit the `S` parameter:

.. doctest-requires:: synphot

   >>> wave = [0.4, 0.5, 0.6] * u.um
   >>> for i in range(len(wave)):
   ...     print('{:5.3e} at {:3.1f}'.format(afrho.fluxd(wave[i], aper, eph),
   ...                                       wave[i]))
   9.162e-14 W / (m2 um) at 0.4 um
   1.048e-13 W / (m2 um) at 0.5 um
   9.641e-14 W / (m2 um) at 0.6 um

To use the Kurucz (1993) model:

.. doctest-requires:: synphot

   >>> from sbpy.spectroscopy.sun import default_sun
   >>> with default_sun.set('Kurucz1993'):            # doctest: +REMOTE_DATA
   ...     print(afrho.fluxd(wave, aper, eph))
   [7.62582935e-14 1.06322888e-13 9.55650074e-14] W / (m2 um)

The `Efrho` class has the same functionality as the `Afrho` class.  The most important difference is that εfρ is calculated using a Planck function and temperature.  `sbpy` follows common practice and parameterizes the temperature as a constant scale factor of $T_{BB} = 278\,r_h^{1/2}$ K, the equilibrium temperature of a large blackbody sphere at a distance $r_h$ from the Sun.

Reproduce the εfρ of 246P/NEAT from Kelley et al. (2013).

  >>> wave = [15.8, 22.3] * u.um
  >>> fluxd = [25.75, 59.2] * u.mJy
  >>> aper = 11.1 * u.arcsec
  >>> eph = {'rh': 4.28 * u.au, 'delta': 3.71 * u.au}
  >>> efrho = Efrho.from_fluxd(wave, fluxd, aper, eph=eph)
  >>> for i in range(len(wave)):
  ...     print('{:5.1f} at {:.1f}'.format(efrho[i], wave[i]))
  396.7 cm at 15.8 um
  421.2 cm at 22.3 um

Compare to 397.0 cm and 424.6 cm listed in Kelley et al. (2013).


Magnitudes
^^^^^^^^^^

`Afrho` and `Efrho` provide `mag` and `from_mag` methods for working with apparent magnitudes.  The default behavior is to use `sbpy`'s calibration framework.

.. doctest-requires:: synphot

  >>> m = 8.49
  >>> aper = 10000 * u.km
  >>> eph = {'rh': 1.45 * u.au,
  ...        'delta': 0.49 * u.au,
  ...        'phase': 17.8 * u.deg}
  >>> afrho = Afrho.from_mag(m, 'vegamag', aper, eph, bandpass='cousins_i',
  ...     phasecor=True)      # doctest: +REMOTE_DATA +IGNORE_OUTPUT
  >>> afrho.value             # doctest: +REMOTE_DATA +FLOAT_CMP
  3423.6675739077887
  >>> afrho.mag('vegamag', aper, eph, bandpass='cousins_i',
  ...     phasecor=True)      # doctest: +REMOTE_DATA +FLOAT_CMP
  8.49


Phase angles and functions
^^^^^^^^^^^^^^^^^^^^^^^^^^

Note the phase angle was not used in the previous section.  The `Afrho` class does not assume any particular phase function when transforming to and from flux density or magnitude.  In order to scale a value to another phase angle, a function is provided.  Returning to the A'Hearn et al. data, we scale Afρ to 0° from to 3.3° phase:

  >>> wave = 5240 * u.AA
  >>> flam = 10**-13.99 * u.Unit('erg/(s cm2 AA)')
  >>> aper = 27200 * u.km
  >>> eph = dict(rh=4.785 * u.au, delta=3.822 * u.au)
  >>> Slam = 1868 * u.W / u.m**2 / u.um
  >>> print(afrho.to_phase(0 * u.deg, 3.3 * u.deg))  # to 0° from 3.3°
  6886.828824340641 cm

The call used the module default phase function.  At the time of writing it is the Halley-Marcus composite phase function (`phase_HalleyMarcus`), but any callable that returns a scale factor from an `astropy` `Quantity` can be used.

  >>> Phi = lambda phase: 10**(-0.016 * phase.to('deg').value)
  >>> print(afrho.to_phase(0 * u.deg, 3.3 * u.deg, Phi=Phi))
  6809.422621373015 cm

An optional parameter, `phasecor`, in the `fluxd` and `from_fluxd` methods indicates if a phase function should be considered.

  >>> eph['phase'] = 3.3 * u.deg       # add phase angle to the ephemeris
  >>> afrho = Afrho.from_fluxd(wave, flam, aper, eph=eph, S=Slam, phasecor=True)
  >>> print('Afρ at 0° phase =', afrho)
  Afρ at 0° phase = 6886.828824340641 cm
  >>> f = afrho.fluxd(wave, aper, eph=eph, S=Slam, phasecor=True)
  >>> print(np.log10(f.to('erg/(s cm2 AA)').value))
  -13.99

Apertures
^^^^^^^^^

Other apertures may be used, as long as they can be converted into an equivalent radius, assuming a coma with a 1/ρ surface brightness distribution.  `sbpy.activity` has a collection of useful geometries.

  >>> from sbpy.activity import CircularAperture, AnnularAperture, RectangularAperture, GaussianAperture
  >>> apertures = (
  ...   ( '10" radius circle', CircularAperture(10 * u.arcsec)),
  ...   (    '5"–10" annulus', AnnularAperture([5, 10] * u.arcsec)),
  ...   (       '2"x10" slit', RectangularAperture([2, 10] * u.arcsec)),
  ...   ('σ=5" Gaussian beam', GaussianAperture(5 * u.arcsec))
  ... )
  >>> print('F_ν = {:.4f}'.format(fnu))
  F_ν = 0.0094 Jy
  >>> for name, aper in apertures:
  ...     afrho = Afrho.from_fluxd(wave, flam, aper, eph, S=Slam)
  ...     print('{:18s} = {:5.0f}'.format(name, afrho))
   10" radius circle =  5917 cm
      5"–10" annulus = 11834 cm
         2"x10" slit = 28114 cm
  σ=5" Gaussian beam =  9442 cm



Reference/API
-------------
.. automodapi:: sbpy.activity
    :no-heading:
