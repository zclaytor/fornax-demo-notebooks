# Cross-matching a user-provided catalog with Pan-STARRS using LSDB

# Learning Goals

By the end of this tutorial, you will:
1. do something
2. do something else

# Introduction

**Explain catalogs used**

The `lsdb` package provides tools to partition a user-provided catalog into HATS format for fast cross-matching. In this notebook, we'll cross-match our own catalog with the Pan-STARRS 1 catalog held in the S3 bucket. The steps are:

1. Upload (or retrieve) a catalog
2. Use `lsdb` to partition our catalog to HATS format
3. Read the PS1 catalog from the S3 bucket
4. Perform the cross-match with `lsdb`

# Runtime

This notebook takes x seconds to run.

# Imports

To get started, we'll need

- `astroquery` to retrieve some catalog data
- `lsdb` for HATS partitioning and cross-matching
- `astropy` for coordinates and units
- `matplotlib` to visualize the result of the cross-match

```python
# Uncomment the next line to install dependencies if needed.
# %pip install -r requirements_user_catalog_xmatch.txt
```


```python
from astroquery.mast import Catalogs
import lsdb
from astropy.coordinates import SkyCoord
import astropy.units as u
import matplotlib.pyplot as plt
```

## 1. Retrieve a catalog

This can be a hosted catalog or from your own file on-disk, as long as it has columns for right ascension and declination. Here we will use `astroquery.mast` to query a subset of the TESS Input Catalog around the northern ecliptic pole.


```python
tic = Catalogs.query_criteria(catalog="TIC", eclat=[85, 90], objType="STAR")
tic = tic["ID", "ra", "dec", "Tmag", "Teff", "lum", "plx"] # downselect some relevant columns
tic
```

## 2. Convert to HATS format

Before cross-matching, we should convert the catalog to HATS format using `lsdb.from_dataframe`. This requires the catalog to be stored as a `pandas` DataFrame, for which `astropy` tables have a built-in conversion method.


```python
tic_hats = lsdb.from_dataframe(tic.to_pandas(), ra_column="ra", dec_column="dec")
tic_hats
```

`lsdb` catalogs appear as lazily loaded tables even when read from in-memory data. We can use `plot_pixels()` to get a quick view of the sky coverage for this catalog section.


```python
tic_hats.plot_pixels()
```

## 3. Open a cloud-based catalog

For our second catalog in the cross-match, we will use the Pan-STARRS 1 survey's second data release. We will load the pre-converted HATS catalog from the cloud.


```python
ps1_path = "s3://stpubdata/panstarrs/ps1/public/hats/otmo"
ps1_margin = "s3://stpubdata/panstarrs/ps1/public/hats/otmo_10arcs"

ps1 = lsdb.open_catalog(
    ps1_path,
    margin_cache=ps1_margin,
    columns=["objName","objID","raMean","decMean"],
)
```

We can also view the sky coverage of the PS1 survey:


```python
ps1.plot_pixels()
```

## 4. Perform the cross-match

We can now perform the cross-match using our HATSified catalog and the pre-formatted cloud catalog. `lsdb` cross-matches are constructed lazily and then explicitly computed.


```python
# Construct cross-match
matched = tic_hats.crossmatch(ps1, radius_arcsec=1, n_neighbors=1, suffixes=("_tic", "_ps1"))
matched
```


```python
%%time
# Perform the cross-match
matched.compute()
```

The cross-match takes a minute or two to complete, and the resulting table has 620793 rows. We can use `plot_points` to view the result of the cross-match on the sky.


```python
# Center our view on the cone.
center = SkyCoord(lon=0*u.deg, lat=90*u.deg, frame="barycentrictrueecliptic").icrs
fov = (10*u.arcminute)

# First the TIC points
matched.plot_points(
    ra_column="ra_tic",
    dec_column="dec_tic",
    center=center,
    fov=fov,
    c="red",
    marker="x",
    s=30,
    label="TIC sources",
)

# Then the PS1 points
matched.plot_points(
    ra_column="raMean_ps1",
    dec_column="decMean_ps1",
    # Can skip the center & fov args here, since this is an overlay
    c="k",
    marker="o",
    s=5,
    label="PS1 sources",
    fov=fov,
)

plt.legend()
```

Pan-STARRS is deeper than the TIC, so every TIC source has a Pan-STARRS match, and the matches are very nearly exact. 

## 5. Enabled Science

This section should explore a science case enabled by this kind of cross-match. A best-case scenario would be to combine PS1 photometry and TESS light curves, using PS1 to extend the TESS light curve baseline. But TESS light curves are not easily loaded with `lsdb`, and going from TIC sources to TESS light curves requires a bit of work using `astroquery`. Open to suggestions. Neither of the catalogs used here are imperative, and either can be swapped out for something more useful. Maybe TESS + Gaia DR3? It would be interesting to do something with Gaia proper motions to propagate them to the TESS observation epoch.
